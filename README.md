## 并发带来的问题
----

### 背景
贝仓业务上用了rocketmq订阅了交易数据，根据订单数据做一系列后续的操作，因为交易那会拆分trade生成多笔订单，所以会产生多笔订单并发到我们系统中，然后业务上给用户发送奖励是按照订单维度。我们系统中用了event/listener来解偶业务逻辑。

### 问题一：
发现这个问题是内测用户反应自己升级之后发送了两次奖励，我们通过数据库查看在同一时间点给用户多发了一次奖励，之后又上elk中查询关于该事件的所有日志发现这个一问题的，所以线上核心业务逻辑一定要学会多记录日志，这样可以追溯发生事件的所有过程
#### 问题根源之一:业务逻辑中先update,之后select取数据导致的问题
一开始开发这块业务的时候并没有考虑到并发的问题，所以代码中先是update了数据之后又去select用户数据，导致select出来的是并发更新之后的结果(对业务逻辑来说这是脏数据)，所以多个线程都会发布等级变化的事件。监听了这个等级变更事件的所有listener都会执行后续操作(其中有个操作就是就是升级奖励,导致的后果就是用户获得多次升级奖励:如发卡券,奖励积分等)。下图展示的是两个(理解了两个之后就可以理解多个的情况)处理线程并发所可能产生的结果。
![avatar](https://github.com/shihuimiao/study-log/blob/master/WechatIMG65.png?raw=true)

代码片段中 可以看出来先执行了changeMemberGrowthValue执行了update之后又去xretailMemberGrowthDao.selectByUid select了用户数据(这里update增加的数据和select的数据是同一张表的同一个字段)导致得到的数据不是这个事件产生的数据(因为拿到字段的值是并发下可能是多个事件update之后的值了)，最后多个线程都发布了等级变化事件(按照业务逻辑只有其中一个线程能触发)。
```java
    public void onApplicationEvent(GrowthValueChangeEvent event) {
        GrowthValueChangeEventBO eventBO = event.getGrowthValueChangeEventBO();

        //记录成长值变更流水
        Long growthWaterId = addMemberGrowthWater(eventBO);

        if (growthWaterId > 0) {
        	//更新用户成长值表中的总成长值
            Integer update = changeMemberGrowthValue(eventBO);
            if (update == 0) {
                //记录错误日志
            }
            //执行更新等级操作
            changeMemberLevel(eventBO,growthWaterId)
        }
    }
    private Integer changeMemberLevel(GrowthValueChangeEventBO eventBO, Long growthWaterId) {
    	//retry为了更新失败设置了重试次数变量
        int retry = 0;
        //更新失败重试三次
        while (retry < 3) {
        	//读取用户当前成长值总数
            MemberGrowthDO memberGrowthDO = memberGrowthDao.selectByUid(eventBO.getUid());
            if (memberGrowthDO == null) {
                return null;
            }

            //计算用户当前等级
            Integer level = GrowthWaterVipLimitMapEnum.getVipByGrowth(memberGrowthDO.getValue());

            //等级没变，不做处理
            if (memberGrowthDO.getLevel().equals(level)) {
                return null;
            }

            //这个version就是更新版本号 数据库更新一次成长值就累加1，更新操作会判断version版本是否与数据库中的version相等，如果相等则执行更新操作(乐观锁)
            Integer version = memberGrowthDO.getVersion();
            Integer updateRes = memberGrowthDao.updateMemberGrowthLevel(eventBO.getUid(), level, version);
            if (updateRes > 0) {
                //发布等级变化的事件
                publishLevelChangeEvent(...));
                return updateRes;
            }
            retry++;
        }
        return null;
    }
```
#### 解决办法:多并发下update添加一个版本号(乐观锁) 然后select在update之前 改变后的值是自己代码来控制
代码片段中 先用xretailMemberGrowthDao.selectByUid(eventBO.getUid()) select数据库中最初的值,然后用updateMemberGrowthByUidWithVersion更新了数据库的数据(使用乐观锁),可以看到业务逻辑自己去计算了newValue的值，而不是从数据库中去select这样就避免了从数据库中读到的数据并不是自己想要的而是对业务来说的脏数据，这里使用乐观锁还有一个原因是考虑到了并发下可能多个线程共同更新数据成功，拿着后续的数据可能又会导致多个线程满足升级的逻辑，所以这里重试可以保证不会出现这种错误产生(因为一个线程读取的数据肯定是用户最新的数据，这时你拿到的用户newlevel已经是升级后的就不会再次去发布等级变更的事件了)。
```java
    private Integer changeMemberGrowth(GrowthValueChangeEventBO eventBO, Long growthWaterId) {
        int retry = 0;
        while (retry < 3) {
            MemberGrowthDO memberGrowthDO = memberGrowthDao.selectByUid(eventBO.getUid());
            if (xretailMemberGrowthDO == null) {
                return 0;
            }

            //获取等级////////////////////////这里等级计算是通过计算得来的//////把更新用户成长值放到了后面
            long newValue = memberGrowthDO.getValue() + eventBO.getValue();

            Integer newlevel = GrowthWaterVipLimitMapEnum.getVipByGrowth(newValue);

            //对于升级成V1以上的用户,保级处罚时不会降级到V0
            if (memberGrowthDO.getLevel() > 0 && newlevel < 1 && (eventBO.getSource() ==                GrowthWaterSourceConstans.SOURCE_LEVEL_TASK_REDUCE)) {
                newlevel = 1;
            }

            Integer version = memberGrowthDO.getVersion();
            /////////////////这里更新了用户等级和成长值///////////////////////
            Integer updateRes = xretailMemberGrowthDao.updateMemberGrowthByUidWithVersion(eventBO.getUid(), newlevel,        eventBO.getValue(), version);
            if (updateRes > 0) {
                //发布等级变化的事件
                if (newlevel > xretailMemberGrowthDO.getLevel()) {
                    publishLevelChangeEvent(new LevelChangeEventBO(....));
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
#### 解决办法:重试次数调大（这里也会有问题，假设拆的单超过重试次数就有机率发生这种问题）最后在重新审视了整个业务逻辑后得到决定了根据trade纬度来执行逻辑,这样我们的业务上订阅了trade用trade来触发我这里的事件,放弃了订阅拆单后的order这样，从根源上避免了产生了这个并发的问题,就不用在后面的业务逻辑来规避这种并发之下产生的种种问题。

### 引申:
- 下图是我以为的高并发下的解决方案
![avatar](https://github.com/shihuimiao/study-log/blob/master/high-concurrency-system-design.png?raw=true)
- 实际上我所遇到的问题是一个很小的点，但引起的问题确是致命的，所以说脱离了业务的设计都是在刷流氓。

### 总结：
- 整个事件经历了三天，让我对并发之后产生的问题有了更多的思考，第一次出现并发问题我们只着眼于产生那个问题的点，没有从全局的角度来审视整个链路的问题导致解决了一个又产生了第二个问题，在之后的工作中要注意从业务着手先看有没有可以规避并发的产生，在没法规避并发的情况下就要仔细的考虑并发后会产生的一系列问题，从而逐个击破。
- 最终我们从源头解决了这个问题,之前没有重视这个问题，以为订阅trade纬度和订阅order纬度对业务来说是一样的。
- 高并发的业务场景下一定要注意代码片段中是否存在线程不安全的地方，如果找到存在线程不安全的方法，则要采取使用锁，这里我们使用了乐观锁来控制并发问题(乐观锁要注意重试次数，至于重试次数设置为多少就看具体的业务来定)，至于为什么不用悲观锁(SELECT ... FOR UPDATE)考虑到性能和系统的吞吐量会下降。
- 通过了这次事件让我体会到了大流量下高并发可能会产生的一系列问题，当你的业务到达一定量级的时候就肯定要开始着手于系统的拆分和整体架构，当然设计架构的时候首要的就是重视那些并发下会产生的问题，一个高可用的系统一定要扛得住大流量的冲击，如何扛住大流量又是另一门学问。

### 最后
- 高并发的业务场景下一定要注意代码片段中是否存在线程不安全的地方，如果找到存在线程不安全的方法，则要采取使用锁，这里我们使用了乐观锁来控制并发问题(乐观锁要注意重试次数，至于重试次数设置为多少就看具体的业务来定)，至于为什么不用悲观锁(SELECT ... FOR UPDATE)考虑到性能和系统的吞吐量会下降
。










