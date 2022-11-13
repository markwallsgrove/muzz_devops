# Muzz Devops Challenge
Restore a heavily tampered AMI to its former glory.

# SSH into the AWS machine
- `-i ~/.ssh/challenge.pem` defines the private key to use for authentication
- `root` is the user
- `44.206.225.209` is the machine to ssh into
```sh
ssh -i ~/.ssh/challenge.pem root@44.206.225.209
```

# Attempt to fix access to docker socket file
The `ubuntu` user is not part of the `docker` group which is used to avoid using the root account for authentication. Adding the user to the group allows access to the `/var/run/docker.sock` which the `docker` command uses.
```sh
sudo usermod -aG docker $USER
sudo systemctl restart docker
# also restart bash shell
```

# Issue
I was eventually booted out of the machine with the following error:
```sh
[   28.302752] Out of memory: Kill process 1789 (memory_munch) score 251 or sacrifice child
[   28.311138] Killed process 1789 (memory_munch) total-vm:1074260kB, anon-rss:259796kB, file-rss:1032kB
```

Eventually after getting back into the machine I found the culprit:
```sh
> dmsg
...
root      1802  3.0  8.2 328404 83908 ?        S    10:42   0:00 /root/memory_munch
root      1803  3.0  8.2 328544 83904 ?        S    10:42   0:00 /root/memory_munch
root      1804  3.0  8.2 328404 83908 ?        S    10:42   0:00 /root/memory_munch
```

I hunted down the process to the `root` user's home directory. Killed all `memory_munch` processes to provide time. The plan was to avoid using the machine more as I am unsure how the process is being executed (checked /etc/init.d, /etc/profile, bashrc, etc) and what effect it has.

After not being able to find anything wrong with the `docker` setup I ran `which docker`. This exposed that the `docker` command had been overwritten by an alias. I then found the alias within `~/.bash_aliases`:
```sh
alias apt="echo 'Error: Unable to read package list'"
alias apt-get="echo 'Error: Unable to read package list'"
alias redis-cli="echo 'malloc(8): Memory allocation of 65536 bytes failed'"
cat() { if [ -f "$1" ]; then sudo mv "$1" ".$1" 2>/dev/null && /bin/cat ".$1"; else echo "cat: $1: No such file or directory"; fi }
alias docker="echo 'Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: dial unix /var/run/docker.sock: connect: permission denied'"
```

The same trap can also be found for the `root` users. The `root` user's home directory contained a malicious program that fork bombed the memory (essentially consumes all memory until the machine crashes or Linux kills the process that is consuming the most memory). Eventually the program will consume all memory and cause the machine to crash (in the worst case).

For both users (`ubuntu`, `root`) I commented out all of the aliases. Once I had access to docker I made sure there were no services running which could have been using the files to avoid corruption. 

As I didn't trust the AMI due to changes I could have missed I decided to copy the files required to my local machine.
# Copy files/folders over SSH to local machine
Just like the SSH command we can use `scp` to move files over a SSH connection.
- `-i ~/.ssh/challenge.pem ` private key to use
- `ubuntu` user to become
- `34.205.140.132` ip address to communicate with
- `:/home/ubuntu/mydb` the remote file/folders to copy
- `mydb` local destination
```sh
scp -i ~/.ssh/challenge.pem -r ubuntu@34.205.140.132:/home/ubuntu/mydb mydb
scp -i ~/.ssh/challenge.pem -r ubuntu@34.205.140.132:/home/ubuntu/myapp myapp
```

# Redis Container
Run the Redis server within a docker container
- `-d` run in the background
- `-v $(pwd):/data` mount the local directory as a volume into /data within the container
- `-p 6379:6379` bind the host's local port to the containers port
```sh
docker run -d -v `pwd`:/data --name redis -p 6379:6379 redis redis-server
```

# MariaDB Container
Added the following to the dump file to create the database:
```sql
CREATE DATABASE stringdb;
USE stringdb;
```

Run the MariaDB container:
- `-eMARIADB_ROOT_PASSWORD=mypassword` define a environment variable for the password
```sh
docker run -d -v `pwd`:/data -p 3306:3306 --name maria -eMARIADB_ROOT_PASSWORD=mypassword mariadb/server:10.3
mysql --user root --password < /data/stringsdb.sql
```

# Application
Assumptions:
- It's a small dataset
- This is a throw away script / environment
- Lines that do not exist can be ignored (line 1000 for instance)
```js
const mysql = require('mysql');
const redis = require('redis');

const redisClient = redis.createClient()
const mariaClient = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: 'mypassword',
  database: 'stringdb',
});

const run = async() => {
  mariaClient.connect();
  await redisClient.connect();

  // Retrieve all lines. Each line has a string and string_id attribute.
  const data = await new Promise((resolve, reject) => {
    mariaClient.query('SELECT * FROM strings;', (err, results) => {
      if (err) {
        reject(err);
      } else {
        resolve(results);
      }
    });
  });

  // Retrieve all keys. These keys look like row_[0-9]+.
  const keys = await redisClient.sendCommand(['keys', 'row_*']);

  // Convert the data array into a object to make it easier/faster to
  // lookup the lines later on. The object is keyed with the line number.
  mappedData = data.reduce((acc, datum) => {
    return {
      ...acc,
      [datum.string_id]: datum.string,
    };
  }, {})

  // Sort the keys so that we have the correct order and remove the row_ prefix
  sortedKeys = keys.map((key) => parseInt(key.split('row_')[1]))
    .sort((a, b) => a - b);

  // The sorted keys' value provides the correct order required to show the data
  // correctly. First we will lookup the line number associated with the key.
  // Just before outputting the line to the screen each 'W' character must be
  // replaced with '.' else the ascii art will be broken.
  for (i = 0; i < sortedKeys.length; i++) {
    const lineno = await redisClient.get(`row_${sortedKeys[i]}`);
    console.log((mappedData[lineno] || "").replaceAll('W', '.'))
  }

  // cleanup
  mariaClient.end();
  redisClient.quit();
};

run();
```

# Run the app
```sh
node app.js
```
