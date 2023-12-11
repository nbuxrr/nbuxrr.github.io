---
title: MySQL binlog 源码
categories: MySQL
tags: [MySQL, binlog]
date: 2021-06-26
---

## 二阶段提交

- prepare阶段
- commit阶段
    - FLUSH阶段
    - SYNC阶段
    - COMMIT阶段

### 数据结构

```c++
// binlog commit的3个阶段的枚举
enum StageID {
FLUSH_STAGE,
SYNC_STAGE,
COMMIT_STAGE,
STAGE_COUNTER
};

// 每个提交阶段都有一个队列(queue<THD*>)
Stage_manager::Mutex_queue Stage_manager::m_queue[STAGE_COUNTER];

class MYSQL_BIN_LOG: public TC_LOG {...}
```

### 调用流程

```c++
// commit主入口
trans_commit
    ha_commit_trans
        // prepare阶段
        MYSQL_BIN_LOG::prepare
            ha_prepare_low
        // commit阶段
        MYSQL_BIN_LOG::commit
            MYSQL_BIN_LOG::ordered_commit
                MYSQL_BIN_LOG::change_stage(thd,
                                            stage: Stage_manager::FLUSH_STAGE 
                                            or Stage_manager::SYNC_STAGE
                                            or Stage_manager::COMMIT_STAGE,
                                            ...)
                    // 将本对象入队列待本阶段的leader处理，followers会一直等在这里，leader会返回到change_stage，进行队列的处理
                    Stage_manager::enroll_for
                        bool leader = m_queue[stage].append(thd);
                        if (!leader) {
                            ...
                            mysql_cond_wait(&m_cond_done, &m_lock_done);
                            ...
                        }
                    //  follower出来条件是leader处理完成
                    MYSQL_BIN_LOG::finish_commit  // 所以follower出来即finish了

                    // leader出来处理队列
                    
                    // FLUSH_STAGE
                    MYSQL_BIN_LOG::process_flush_stage_queue
                        THD *first_seen = m_queue[stage].fetch_and_empty()          // 将整个queue取出
                        MYSQL_BIN_LOG::assign_automatic_gtids_to_flush_group(first_seen)         、、
                    // SYNC_STAGE

                    // COMMIT_STAGE

```




