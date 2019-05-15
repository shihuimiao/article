## 并发带来的问题
----

### 背景
贝仓业务上用了rocketmq来获取交易那的订单数据，因为交易那会拆分trade生成多笔订单，所以会产生多笔订单并发到系统中，然后业务上给用户发送奖励是按照订单纬度。业务中用了event/listener来结偶逻辑。

### 问题一：
#### 问题根源之一:业务逻辑中update又重新去select取数据
最开始开发这块逻辑的时候没有考虑到并发的问题，所以先update之后又去select了用户的数据，导致select出来的是两次更新之后的结果，所以两个线程都会发布等级变化的事件。之后监听了这个事件的所有listener都会执行相应的操作。
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG65.png?raw=true)

代码片段中 可以看出来先执行了changeMemberGrowthValue执行了update之后又去xretailMemberGrowthDao.selectByUid select了用户数据导致得到的数据是不是这个事件产生的数据 而是并发update之后的数据，最后两个线程都发布了等级变化的事件
```java
        public void onApplicationEvent(GrowthValueChangeEvent event) {
        GrowthValueChangeEventBO eventBO = event.getGrowthValueChangeEventBO();

        //判断是否存在uid
        XretailMemberDO xretailMemberDO = xretailMemberDao.selectByUid(eventBO.getUid());

        if (xretailMemberDO == null) {
            logger.info("process growth value change event fail:{},member is not exists", eventBO);
            return;
        }

        logger.info("process growth value change event:{}", eventBO);

        Long growthWaterId = addMemberGrowthWater(eventBO);

        if (growthWaterId > 0) {
            Integer update = changeMemberGrowthValue(eventBO);
            if (update == 0) {
                logger.info("process update growth failed:{}", eventBO);
            }
            changeMemberLevel(eventBO, growthWaterId);
        }
    }
    private Integer changeMemberLevel(GrowthValueChangeEventBO eventBO, Long growthWaterId) {
        int retry = 0;
        while (retry < 3) {
            XretailMemberGrowthDO xretailMemberGrowthDO = xretailMemberGrowthDao.selectByUid(eventBO.getUid());
            if (xretailMemberGrowthDO == null) {
                return null;
            }

            //获取等级
            Integer level = GrowthWaterVipLimitMapEnum.getVipByGrowth(xretailMemberGrowthDO.getValue());

            //等级没变，不做处理
            if (xretailMemberGrowthDO.getLevel().equals(level)) {
                return null;
            }

            //对于升级成V1以上的用户,保级处罚时不会降级到V0
            if (xretailMemberGrowthDO.getLevel() > 0 && level < 1 && (eventBO.getSource() == GrowthWaterSourceConstans.SOURCE_LEVEL_TASK_REDUCE)) {
                level = 1;
            }

            Integer version = xretailMemberGrowthDO.getVersion();
            Integer updateRes = xretailMemberGrowthDao.updateMemberGrowthLevel(eventBO.getUid(), level, version);
            if (updateRes > 0) {
                //发布等级变化的事件
                publishLevelChangeEvent(new LevelChangeEventBO(eventBO.getUid(), xretailMemberGrowthDO.getLevel(), level, DateUtils.getNowTime(), growthWaterId, eventBO.getSource()));
                return updateRes;
            }
            retry++;
        }
        return null;
    }
```
#### 解决办法:多并发下update添加一个版本号(相当于添加了乐观锁) 然后select在update之前 改变后的值是自己代码来控制
代码片段中 先用xretailMemberGrowthDao.selectByUid(eventBO.getUid()) select数据库中最初的值,然后用updateMemberGrowthByUidWithVersion更新了数据库的数据(使用乐观锁),可以看到业务逻辑自己去计算了newValue的值，没有从数据库中读取，这就可以避免从数据库中得到不是这个线程想要的那个值了
```java
private Integer changeMemberGrowth(GrowthValueChangeEventBO eventBO, Long growthWaterId) {
        int retry = 0;
        while (retry < 3) {
            XretailMemberGrowthDO xretailMemberGrowthDO = xretailMemberGrowthDao.selectByUid(eventBO.getUid());
            if (xretailMemberGrowthDO == null) {
                return 0;
            }

            //获取等级
            long newValue = xretailMemberGrowthDO.getValue() + eventBO.getValue();

            Integer newlevel = GrowthWaterVipLimitMapEnum.getVipByGrowth(newValue);

            //对于升级成V1以上的用户,保级处罚时不会降级到V0
            if (xretailMemberGrowthDO.getLevel() > 0 && newlevel < 1 && (eventBO.getSource() == GrowthWaterSourceConstans.SOURCE_LEVEL_TASK_REDUCE)) {
                newlevel = 1;
            }

            Integer version = xretailMemberGrowthDO.getVersion();
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

        return null;
    }
```

### 问题二：
#### 问题根源之一:乐观锁重试次数过小
导致最终数据没有更新到数据库中，从而没有发生用户等级变更事件，之后的数据对用户产生了很大的影响
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG66.png?raw=true)
#### 解决办法:重试次数调大（这里也会有问题，假设拆的单超过重试次数就有机率发生这种问题）最终决定业务上根据trade纬度来执行逻辑，从根源上避免了产生了并发的问题

### 总结：
整个事件经历了三天事件，
高并发的业务场景下一定要注意代码片段中是否存在线程不安全的地方，如果找到存在线程不安全的方法，则要采取使用锁，这里我们使用了乐观锁来控制并发问题(乐观锁要注意重试次数，至于重试次数设置为多少就看具体的业务来定)，至于为什么不用悲观锁(SELECT ... FOR UPDATE)考虑到性能和系统的吞吐量会下降。 
