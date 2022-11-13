# Muzz Devops Challenge

restore a heavily tampered AMI to its former glory.

# SSH into the AWS machine using a certain private key and username
```sh
ssh -i ~/.ssh/challenge.pem root@44.206.225.209
```

# Attempt to fix access to docker socket file
```sh
sudo usermod -aG docker $USER
sudo systemctl restart docker
# also restart shell
```

Attempting to find any permission issues with folder structure
```sh
find / -name docker 2>/dev/null | xargs -I{} ls -la {}
```

# Issue
Was booted out of the machine:
```sh
[   28.302752] Out of memory: Kill process 1789 (memory_munch) score 251 or sacrifice child
[   28.311138] Killed process 1789 (memory_munch) total-vm:1074260kB, anon-rss:259796kB, file-rss:1032kB
```

Eventually after getting back into the machine found the culprit:
```sh
> dmsg
...
root      1802  3.0  8.2 328404 83908 ?        S    10:42   0:00 /root/memory_munch
root      1803  3.0  8.2 328544 83904 ?        S    10:42   0:00 /root/memory_munch
root      1804  3.0  8.2 328404 83908 ?        S    10:42   0:00 /root/memory_munch
```

Hunted down the process to the root user's home directory. Killed all `memory_munch` processes to provide time. Avoid using the machine any more as I am unsure how the process is being executed (checked /etc/init.d, /etc/profile, bashrc, etc).

spiked `.bash_aliases` file:
```sh
alias apt="echo 'Error: Unable to read package list'"
alias apt-get="echo 'Error: Unable to read package list'"
alias redis-cli="echo 'malloc(8): Memory allocation of 65536 bytes failed'"
cat() { if [ -f "$1" ]; then sudo mv "$1" ".$1" 2>/dev/null && /bin/cat ".$1"; else echo "cat: $1: No such file or directory"; fi }
alias docker="echo 'Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: dial unix /var/run/docker.sock: connect: permission denied'"
```

Same trap for `root` and `ubuntu` users. The `root` user's home directory contained a malicious program that fork bombed the memory (essentially consumes all memory until the machine crashes). Eventually the program will consume all memory and cause the machine to crash.

# Copy files/folders over SSH to local machine
```sh
scp -i ~/.ssh/challenge.pem -r ubuntu@34.205.140.132:/home/ubuntu/mydb mydb
scp -i ~/.ssh/challenge.pem -r ubuntu@34.205.140.132:/home/ubuntu/myapp myapp
```

# Redis Container
Run the Redis container
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
```sh
docker run -d -v `pwd`:/data -p 3306:3306 --name maria -eMARIADB_ROOT_PASSWORD=mypassword mariadb/server:10.3
mysql --user admin_restore --password < /data/stringsdb.sql
```

# Application
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

  const data = await new Promise((resolve, reject) => {
    mariaClient.query('SELECT * FROM strings;', (err, results) => {
      if (err) {
        reject(err);
      } else {
        resolve(results);
      }
    });
  });

  const keys = await redisClient.sendCommand(['keys', 'row_*']);

  mappedData = data.reduce((acc, datum) => {
    return {
      ...acc,
      [datum.string_id]: datum.string,
    };
  }, {})

  sortedKeys = keys.map((key) => parseInt(key.split('row_')[1]))
    .sort((a, b) => a - b);

  for (i = 0; i < sortedKeys.length; i++) {
    const value = await redisClient.get(`row_${sortedKeys[i]}`);
    console.log((mappedData[value] || "").replaceAll('W', '.'))
  }

  mariaClient.end();
  redisClient.quit();
};

run();
```

# Run the app
```sh
node app.js
```
