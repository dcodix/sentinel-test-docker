# sentinel-test-docker

Instructions for firing 2 (or more) redis+sentinel instances with Docker to "taste" sentinel failover.

## Building Docker image
```
cd tmp
git clone git@github.com:dcodix/sentinel-test-docker.git
cd sentinel-test-docker
cd docker-redis
docker build -t redis_sentinel .
```

## Running Docker image
```
docker run -i -t redis_sentinel
```

## Sentinel failover test instructions

The test will consist on launching 2 redis+sentinel instances, configure redis and sentinel on both instances, and test the failover.
During the next steps I will be refering to the 2 instances as "T1" and "T2" as we will be opening 2 terminals.
We can do all tests from the bash prompt in both docker instances but if prefered we could install redis in our machine or launch a third instance to use redis-cli remotely.

### Run 2 docker redis instances
After this step we will have 2 terminals with bash running redis and sentinel.
#### T1
```
alias docker="sudo docker"
docker run -i -t redis_sentinel
```
#### T2
```
alias docker="sudo docker"
docker run -i -t redis_sentinel
```

### Clean default configuration
We clean the default configuration and check that both redis are "master" and that sentinel has no master configured.
We do the same in both terminals:
#### Remove default master from sentinel and check
```
redis-cli -p 26379 sentinel remove mymaster
redis-cli -p 26379 sentinel masters
```

#### Check that redis is master
```
redis-cli info replication
```

### Check there is no data and add some (different in both instances)
#### T1
```
redis-cli keys "*"
redis-cli set dr who
redis-cli keys "*"
```
#### T2
```
redis-cli keys "*"
redis-cli set rose tyler
redis-cli keys "*"
```

### Get the IP on T1 and configure sentinels
We are going to configure the T1 redis instance as master in both sentinels, so in T1 and T2, supposing the IP is 172.17.0.24 we do:
```
redis-cli -p 26379 sentinel monitor mymaster 172.17.0.24 6379 2
```
And we check:
```
redis-cli -p 26379 sentinel masters
```
If we add the redis master in one of the sentinels and we check before adding it to the second sentinel we can see the lines:
```
...
   31) "num-other-sentinels"
   32) "0"
...
```
After adding the redis master to the second sentinel we can see:
```
...
   31) "num-other-sentinels"
   32) "1"
...
```
So, YES!, the sentinels autodiscover other sentinels.

### Add the slave
For now we have 2 sentinels monitoring a standalone redis master, there are no slaves. We can check it in any of the instances by:
```
redis-cli -p 26379 sentinel slaves mymaster
```
Its time to add a salve so in T2 we do:
```
redis-cli slaveof 172.17.0.24 6379
```
Now, the slave will sync with the master and we can check it will have the same data that master (and lost it's own old data):
```
redis-cli info replication
redis-cli keys "*"
```
Now both sentinels would have to autodetect the new slave and it would be eligible for the failover:
```
redis-cli -p 26379 sentinel slaves mymaster
```

### Time to test the failover
On T1 we kill redis.
On any of the sentinels we can observe:
```
redis-cli -p 26379 sentinel masters
```
We will be seeing how a time counter is growing till it will arrive to 30s (default), moment in which it will failover and we will see the new master. In that moment we can check in T2 to see it is now master:
```
redis-cli info replication
```

### Launching the fallen redis
It is time to launch the fallen redis again to see what happens.
First we will add some new data to the master in T2:
```
redis-cli set amy pond
redis-cli keys "*"
```
On T1:
```
/usr/local/bin/redis-server /etc/redis/redis.conf &
```
After some seconds we check T1:
```
redis-cli info replication
redis-cli keys "*"
```
It automatically started as slave of the T2 master and the system is "ready to fail again" ;-)
