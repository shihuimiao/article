## 并发带来的问题
----

### 背景
贝仓业务上用了rocketmq来获取交易那的订单数据，因为交易那会拆分trade生成多笔订单，所以会产生多笔订单并发到系统中，然后业务上给用户发送奖励是按照订单纬度。业务中用了event/listener来结偶逻辑。

### 问题一：
#### 问题根源之一:业务逻辑中先去update再重写select去取数据
最开始开发这块逻辑的时候没有考虑到并发的问题，所以先update之后又去select了用户的数据，导致select出来的是两次更新之后的结果，所以两个线程都会发布等级变化的事件。之后监听了这个事件的所有listener都会执行相应的操作。
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG65.png?raw=true)

- 代码片段中 可以看出来先执行了changeMemberGrowthValue执行了update之后又去xretailMemberGrowthDao.selectByUid select了用户数据导致得到的数据是不是这个事件产生的数据 而是并发update之后的数据，最后两个线程都发布了等级变化的事件
```java
        Long growthWaterId = addMemberGrowthWater(eventBO);

        Integer update = changeMemberGrowthValue(eventBO);
        if (update == 0) {
            logger.info("process update growth failed:{}", eventBO);
        }
        int retry = 0;
        while (retry < 3) {
            XretailMemberGrowthDO xretailMemberGrowthDO = xretailMemberGrowthDao.selectByUid(eventBO.getUid());
            if (xretailMemberGrowthDO == null) {
                return 0;
            }
            if (xretailMemberGrowthDO.getLevel() > 0 && newlevel < 1 && (eventBO.getSource() == GrowthWaterSourceConstans.SOURCE_LEVEL_TASK_REDUCE)) {
                newlevel = 1;
            }
            Integer updateRes = xretailMemberGrowthDao.updateMemberGrowthByUidWithVersion(eventBO.getUid(), newlevel, eventBO.getValue(), version);
            if (updateRes > 0) {
                //发布等级变化的事件
                if (newlevel > xretailMemberGrowthDO.getLevel()) {
                    publishLevelChangeEvent(new LevelChangeEventBO(eventBO.getUid(), xretailMemberGrowthDO.getLevel(), newlevel, DateUtils.getNowTime(), growthWaterId, eventBO.getSource()));
                }
                return 1;
            }
            retry++;
        }
```
#### 解决办法:多并发下update添加一个版本号(相当于添加了乐观锁) 然后select在update之前 改变后的值是自己代码来控制

### 问题二：
#### 问题根源之一:乐观锁重试次数过小
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG66.png?raw=true)
#### 解决办法:重试次数调大（这里也会有问题，假设拆的单超过重试次数就有机率发生这种问题）最终决定业务上根据trade纬度来执行逻辑，从根源上避免了产生了并发的问题













        
        
        
        
        
        
