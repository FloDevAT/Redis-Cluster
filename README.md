# 🚀 Redis Cluster

## 📌 Introduction

This repository contains a brief documentation of my Redis Cluster exercise. The following steps were completed in this project:

- ✅ **Setting up a Redis Cluster**
- ✅ **Configuring Geographic Partitioning**
- ✅ **Conducting Failover and Performance Tests**

---

## 🛠️ Step 1: Initial Setup

To create a realistic lab environment, I used Docker to simulate multiple servers in a network, each running a Redis instance. To keep things simple, I designed a basic network structure, documented below:

### 🌍 IP Addressing Schema

| **Instance Name**       | **IP Address** |
|-------------------------|---------------|
| `redis_master_asia`    | `10.0.10.1`   |
| `redis_master_eu`      | `10.0.10.2`   |
| `redis_master_us`      | `10.0.10.3`   |
| `redis_replicant_asia` | `10.0.10.4`   |
| `redis_replicant_eu`   | `10.0.10.5`   |
| `redis_replicant_us`   | `10.0.10.6`   |

---

### ⚡ Creating the Cluster

Normally, `redis-cli` should automatically assign the correct master-replica pairs, but in my case, it didn't, so I configured them manually:

```sh
redis-cli --cluster create 10.0.10.1:7000 10.0.10.2:7000 10.0.10.3:7000 --cluster-yes
redis-cli --cluster add-node 10.0.10.4:7000 10.0.10.1:7000 --cluster-slave
redis-cli --cluster add-node 10.0.10.5:7000 10.0.10.2:7000 --cluster-slave
redis-cli --cluster add-node 10.0.10.6:7000 10.0.10.3:7000 --cluster-slave
```

To verify the cluster status, use:

```sh
redis-cli -p 7000 cluster nodes
```

---

## 🔍 Slot Verification & Issue Identification

To ensure our keys are correctly distributed across nodes, we use the `{eu}`, `{us}`, and `{asia}` prefixes.

```sh
redis-cli -p 7000 CLUSTER KEYSLOT {asia}:test
redis-cli -p 7000 CLUSTER KEYSLOT {eu}:test
redis-cli -p 7000 CLUSTER KEYSLOT {us}:test
```

Example output:

```
(integer) 8971 # {asia}
(integer) 6893 # {eu}
(integer) 14680 # {us}
```

To check which node is responsible for each slot, run:

```sh
redis-cli -p 7000 -c CLUSTER SLOTS
```

### 📊 Slot Distribution Table

| **Node**               | **Slot Range** |
|------------------------|---------------|
| `redis_master_asia`   | `0 - 5460`    |
| `redis_master_eu`     | `5461 - 10922`|
| `redis_master_us`     | `10923 - 16383`|

However, based on the key slot values, there is a misalignment:

```
(integer) 8971      -> redis_master_eu (❌ Should be in redis_master_asia)
(integer) 6893      -> redis_master_eu (✅ Correct)
(integer) 14680     -> redis_master_us (✅ Correct)
```

---

## 🔧 Fixing Slot Alignment

To fix this issue, we need to migrate the slots to their correct nodes.

### 🔹 Master Node Identifiers

```
redis_master_asia -> cca08c426f25595c7fa0bde1f5e2b4ea0ab118a8
redis_master_eu -> c33e50b331faf63597d9267f28f8e75edf0a861e
redis_master_us -> 8a6324404e5e64bffa32a6c5e705283c896f6bc7
```

### 🔹 Commands to Migrate Slots

```sh
redis-cli -p 7000 -c CLUSTER SETSLOT 8971 IMPORTING cca08c426f25595c7fa0bde1f5e2b4ea0ab118a8 # on redis_master_asia
redis-cli -p 7000 -c CLUSTER SETSLOT 8971 MIGRATING c33e50b331faf63597d9267f28f8e75edf0a861e # on redis_master_eu
redis-cli -p 7000 -c CLUSTER SETSLOT 8971 NODE cca08c426f25595c7fa0bde1f5e2b4ea0ab118a8 # on redis_master_asia
```

---

## 🏆 Verification: Key Distribution Test

Now, we will test whether the keys are correctly placed in their respective nodes.

```sh
redis-cli -p 7000 -c SET {eu}:demo "this_is_a_demo"
redis-cli -p 7000 -c SET {asia}:demo "this_is_a_demo"
redis-cli -p 7000 -c SET {us}:demo "this_is_a_demo"
```

To check key placement, run:

```sh
redis-cli -p 7000 -c KEYS *demo*
```

### ✅ Expected Outputs

#### **On `redis_master_us`**
```sh
redis-cli -p 7000 -c KEYS *demo*
1) "{us}:demo"
```

#### **On `redis_master_eu`**
```sh
redis-cli -p 7000 -c KEYS *demo*
1) "{eu}:demo"
```

#### **On `redis_master_asia`**
```sh
redis-cli -p 7000 -c KEYS *demo*
1) "{asia}:demo"
```

---

## 👁 Sentinel Setup

Now we setup three different sentinel instances (one for each region, even though they will supervise all instances). The configuration file for the sentinel instances looks like this:

```
port 5000

sentinel monitor redis_master_asia 10.0.10.1 7000 2
sentinel monitor redis_master_eu 10.0.10.2 7000 2
sentinel monitor redis_master_us 10.0.10.3 7000 2

sentinel down-after-milliseconds redis_master_asia 5000
sentinel failover-timeout redis_master_asia 10000
sentinel parallel-syncs redis_master_asia 1

sentinel down-after-milliseconds redis_master_eu 5000
sentinel failover-timeout redis_master_eu 10000
sentinel parallel-syncs redis_master_eu 1

sentinel down-after-milliseconds redis_master_us 5000
sentinel failover-timeout redis_master_us 10000
sentinel parallel-syncs redis_master_us 1
```

Using my sandbox it is enough to also just run the docker-compose script, since
it will add the sentinel versions anyway. When you stop a master redis instance
you can see in the logs of the sentinel instance, that it gave the role to the master e.g.


## 🎯 Conclusion

With these configurations and tests, we now have a fully functional Redis Cluster with **proper geographic key distribution**. 🚀🎉

---

💡 *This documentation serves as a guide for setting up and troubleshooting Redis Clusters with geographic partitioning.*
