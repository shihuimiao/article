## 并发带来的问题
----

### 背景
贝仓业务上用了rocketmq来获取交易那的订单数据，因为交易那会拆分trade生成多笔订单，所以会产生多笔订单并发到系统中，然后业务上给用户发送奖励是按照订单纬度。业务中用了event/listener来结偶逻辑。

### 问题根源之一:业务逻辑中先去update再重写select去取数据
最开始开发这块逻辑的时候没有考虑到并发的问题，所以先update之后又去select了用户的数据，导致select出来的是两次更新之后的结果，所以两个线程都会发布等级变化的事件。之后监听了这个事件的所有listener都会执行相应的操作。
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG65.png?raw=true)

```java
        Long growthWaterId = addMemberGrowthWater(eventBO);

        if (growthWaterId > 0) {
            Integer update = changeMemberGrowthValue(eventBO);
            if (update == 0) {
                logger.info("process update growth failed:{}", eventBO);
            }
            changeMemberLevel(eventBO, growthWaterId);
        }
        
        private Integer changeMemberGrowth(GrowthValueChangeEventBO eventBO, Long growthWaterId) {
        int retry = 0;
        while (retry < 20) {
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

