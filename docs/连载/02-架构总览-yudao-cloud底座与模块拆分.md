# 架构总览：yudao-cloud底座与模块拆分

上一章我讲了为什么选 genchuan-smart-city 做二开底座。这一章把镜头拉高，讲清楚整个项目的架构骨架：yudao-cloud 微服务底座是怎么组织的，产业园区模块在这张大图里占什么位置，以及它的分层约定为什么值得你原样继承。理解了这层骨架，后面八章的每一段业务代码才有落脚点。

## 先看全景：这个仓库里到底住了多少东西

把仓库 clone 下来，根目录大致是这样一幅景象：`yudao-dependencies`（统一依赖版本管理）、`yudao-framework`（框架核心封装）、`yudao-gateway`（网关）、`yudao-server`（单体启动器）、一串 `yudao-module-*`（system 系统管理、infra 基础设施、bpm 工作流、pay 支付、mall 商城、ai 大模型等），以及 genchuan 团队自己加的一排业务模块——`genchuan-module-chargepark`（充电场站）、`genchuan-module-datahub`（数据中枢）、`genchuan-module-education-teaching`（教育）、`genchuan-module-urban`（城市治理）、`genchuan-module-wateraffairs`（水务），当然还有我们的主角 `genchuan-module-industrialpark`（产业园区）。

第一次看到这个目录结构时，我的感受是"夸张"——这是一个把智慧城市各个垂直行业塞进一个仓库的做法。但对二开者来说，这种"全家桶"反而友好：**模块之间靠 Maven 依赖和 RPC 接口解耦，你完全可以只编译、只部署自己需要的那几个**。我改造时实际启用的服务清单是：Nacos、Gateway、system 模块（用户权限基础）、infra 模块（文件、定时任务）、industrialpark 模块，加上 MySQL 和 Redis。其余十几个业务模块一行代码没动、一个服务没起。

这里要给 yudao 框架记一功。它的 `yudao-dependencies` 把全仓库的依赖版本锁在一个父 POM 里，Spring Boot、Spring Cloud、MyBatis-Plus、Redisson 的版本号只在最顶层出现一次。我升级一个工具库版本时，只需要改一处，全仓库生效——这种版本治理在多模块项目里是保命的设计，没有它，十几个模块各自引依赖，Spring 的类冲突能把人逼疯。

## 产业园区模块的内部地形

`genchuan-module-industrialpark` 自己又拆了四个子模块，我改造前统计过一遍代码量（Java 文件数）：

- **accessmgmt（出入管理）**：127 个文件，四个子模块里最重的。内部再分人脸门禁（facemgmt/faceaccess）、车辆通行（vehicleaccess）、访客三件套（visitormgmt/visitoraccess/visitorappoint）、停车三件套（parkingspace/parkingmgmt/parkingpayment），以及通行记录与报表（accessrecord/accessreport/cyclereport）；
- **energymgmt（能耗管理）**：30 个文件，数据采集（datacollect）、能耗监测（energymonitor）、区域监测（areamonitor）三块；
- **enterprisesvc（企业服务）**：26 个文件，企业档案（enterprisefile）和员工管理（staffmgmt）；
- **integratedsecurity（综合安防）**：16 个文件，视频监控（videomonitor）与实时监测（realtimemonitor）。

每个子模块内部再拆 `-api` 和 `-server` 两层：`-api` 放对外暴露的 RPC 接口定义和数据传输对象，给其他模块远程调用用；`-server` 放真正的实现。这是芋道微服务版的标配拆法，好处是模块间依赖只指向对方的 api 包，实现类的改动不会传染编译错误。

出入管理为什么这么重？你站在园区运营的视角想就明白了：一个园区一天之内发生频率最高的动作就是"人和车过闸机"——员工刷脸上班、访客预约进门、车辆识别入场、临停车扫码缴费。这四类动作共享同一套底层设施（闸机设备、通行记录表、放行策略），自然都归到 accessmgmt 一个子模块里管。

## 分层约定：六层结构和它们的职责边界

翻开任意一个业务包，比如访客预约，目录结构都是这样的：

```
module/accessmgmt/
├── controller/admin/visitormgmt/visitorappoint/
│   ├── VisitorAppointController.java      # REST 接口
│   └── vo/                                 # 请求/响应对象
│       ├── VisitorAppointPageReqVO.java
│       ├── VisitorAppointRespVO.java
│       └── VisitorAppointSaveReqVO.java
├── service/visitormgmt/visitorappoint/
│   ├── VisitorAppointService.java         # 接口
│   └── VisitorAppointServiceImpl.java     # 实现
└── dal/
    ├── dataobject/visitormgmt/visitorappoint/
    │   └── VisitorAppointDO.java          # 表映射对象
    └── mysql/visitormgmt/visitorappoint/
        └── VisitorAppointMapper.java      # 数据访问
```

这套结构的职责边界划得非常清楚，我逐层说一遍，并标注二开时最容易犯错的位置。

**Controller 层**只做三件事：声明路由、打权限注解、调 Service。业务逻辑出现在 Controller 里是这个代码库的"死罪"——我看过它所有的 Controller，最长的也只是参数组装。这样 Controller 就薄到可以不看，所有业务真相都在 Service 层。

**VO 层**按用途拆分：`PageReqVO` 管分页查询条件、`SaveReqVO` 管新增和修改入参、`RespVO` 管返回体。入参对象上挂满 `@NotNull`、`@NotEmpty` 这类校验注解，配合 Service 类上的 `@Validated`，参数校验在进入业务代码之前就完成了。二开时图省事复用 RespVO 当入参，是这个结构里最常见的破窗——一旦开始，字段校验就会逐渐失守。

**Service 层**是唯一的业务真相所在地。接口类定义契约，实现类写逻辑，事务边界也在这里用 `@Transactional(rollbackFor = Exception.class)` 声明。后面章节里的状态机、计费规则，全部长在这一层。

**DO 层**映射表结构，**Mapper 层**继承芋道封装的 `BaseMapperX`，普通 CRUD 一行不写，复杂查询用 QueryWrapper 或 XML。拿能耗采集的 DO 举一段原文：

```java
@TableName("energy_collect")
@KeySequence("energy_collect_seq") // 用于 Oracle、PostgreSQL、Kingbase、DB2、H2 数据库的主键自增。如果是 MySQL 等数据库，可不写。
@Data
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EnergyCollectDO extends BaseDO {

    @TableId
    private Long id;
    /**
     * 设备类型：电表/水表/气表
     */
    private String deviceType;
    /**
     * 能耗类型：电/水/气/热
     */
    private String energyType;
    /**
     * 采集时间
     */
    private LocalDateTime collectTime;
    /**
     * 采集状态：采集正常/采集异常
     */
    private String collectStatus;
    /**
     * 能耗数值
     */
    private BigDecimal energyValue;
    /**
     * 采集频率，单位分钟
     */
    private Integer collectFreq;
}
```

`@KeySequence` 这个注解值得单独说。它的存在意味着这套代码的设计者认真考虑过"换数据库"的问题：主键自增在 MySQL 里靠 `AUTO_INCREMENT`，但在 Oracle、PostgreSQL 里要走 sequence，框架在 INSERT 前动态取序列值补齐主键。我的生产环境用的就是 PostgreSQL，这个注解让业务代码一行不改就完成了数据库迁移——这不是过度设计，这是园区行业客户经常指定国产数据库的现实需求。

## BaseDO：所有表的公共底座

上一章贴的 `VisitorAppointDO` 和刚才的 `EnergyCollectDO` 都继承自 `BaseDO`。这个基类定义在 `yudao-framework` 里，包含五个字段：`creator`（创建人）、`createTime`、`updater`（更新人）、`updateTime`、`deleted`（逻辑删除标记）。

配合框架的 MyBatis-Plus 配置，这五个字段的行为是全自动的：插入时自动填充创建人和时间，更新时自动刷更新人和时间，`deleted = true` 的记录在所有查询里自动过滤。业务代码里从头到尾看不到一句 `SET update_time = now()`。

这套机制带来的不只是省代码，更是一致性。我审查过不少自研项目的库表，十个项目里有八个的时间字段填充逻辑各不相同——有的应用层填、有的数据库 DEFAULT 填、有的干脆忘了填。排查问题时，"这条记录到底是谁、什么时候改的"成了悬案。改造这套底座之后，全库的审计口径天然统一。

逻辑删除也值得多说一句。`deleted` 字段配合 MyBatis-Plus 的逻辑删除插件，DELETE 语句被自动改写成 UPDATE deleted 标记。园区业务里有大量"不能真删"的数据——通行记录是安防追溯的证据、账单记录是财务对账的凭证，逻辑删除让误操作可恢复，也让历史数据在需要时仍可查询。当然它也埋着一个坑：逻辑删除的表，唯一索引和已删除记录会打架（比如车位编号 unique 约束下，删了的车位再建同编号的会冲突），这个坑我在第 5 章讲车位台账时会展开。

## 多租户：框架底座里的隔离设计

园区平台的客户是"园区运营方"，一个平台未来很可能同时服务多个园区甚至多个运营主体——这就要求系统从第一天就有租户隔离的机制。芋道框架内置了多租户支持，这也是我选它的重要加分项，值得单独讲一讲。

它的实现思路是**共享库加租户字段**：所有业务表都带一个 `tenant_id` 列，框架通过 MyBatis-Plus 的 TenantLineInnerInterceptor 在 SQL 解析层自动给每条查询、插入追出租户条件——业务代码里完全看不到 tenant_id，防泄露却无处不在。上下文里的租户标识来自登录令牌，请求进来时由过滤器解析并注入 ThreadLocal。

这套机制对二开的两个直接影响，我都是在第一周就确认清楚的。

第一，**建新表必须带 tenant_id 并注册到租户白名单的配置逻辑里**。框架维护了一张"哪些表参与租户隔离"的清单，新表若不在清单里，拦截器跳过它——数据就会在租户间裸奔。我二开新增的十几张表，建表脚本里统一带上 tenant_id bigint not null default 0，同时补注册，一步不落。

第二，**有一些表必须豁免租户隔离**。比如系统级的字典表、区域配置表，本来就是全租户共享的，不豁免的话每条查询都白加一个无意义的过滤条件。框架用注解或在配置里声明忽略表名，语义清楚。判断一张表"隔离还是共享"的标准是数据归属：租户产生并拥有的数据隔离，平台定义的数据共享。

多园区场景的真实需求里还有一层"跨租户查询"——集团总部要看旗下所有园区的汇总报表。框架提供了绕过租户过滤的显式开关（一个上下文方法，用完即恢复），我的做法是把这类汇总查询收敛到一个专门的报表服务里，普通业务接口永远不允许跨租户，避免开关滥用把隔离墙打出洞。

## 网关与鉴权：一次请求的完整安检链

第 2 章开头我给过 URL 约定，这里把请求穿过网关之后的完整安检链补上，排查权限问题时你会需要这张地图。

请求到达 Gateway，第一站是路由匹配——按 `/admin-api/{模块}` 前缀找到下游服务并转发，这一层只认路径不认人。第二站在下游服务的过滤器链：先验令牌（框架默认用访问令牌，存 Redis，带过期时间），令牌有效则解析出用户 ID、租户 ID 注入上下文；无效则直接拦下返回 401。

第三站是方法级的权限校验。芋道用自定义注解 `@PreAuthorize`（基于 Spring Security 扩展）声明每个接口需要的权限字符串，形如 `accessmgmt:visitor-appoint:audit`——模块、业务、动作三段式。权限串与角色、菜单的关联关系存在 system 模块，校验时远程查一次（有缓存）。这就是为什么每个 Controller 方法头上都挂着一行注解：它不只是文档，是真实的门禁。

我建议每个二开者做新接口时都把权限串当成第一公民来设计：先定权限串，再配菜单与角色，最后写实现。顺序反了的话，功能上线了权限配不齐，要么全员可见（事故），要么无人可用（工单）。

## 模块启动：一个最小可运行的组合

架构讲完，落地一下。二开期间我本地开发时起的最小服务组合是：

- **Nacos**：注册中心 + 配置中心，模块服务启动时把地址注册上去，Gateway 从这里发现下游；
- **Gateway**：统一入口，路由规则把 `/admin-api/accessmgmt/**` 转发到出入管理服务、`/admin-api/energymgmt/**` 转发到能耗服务；
- **system 服务**：用户、角色、权限、租户，所有业务模块的权限校验都依赖它签发的令牌；
- **infra 服务**：文件存储、定时任务、API 日志，能耗采集的定时调度靠它；
- **industrialpark 服务**：我们的业务主战场；
- **MySQL + Redis**：存储与缓存。

URL 约定也顺带说一下，排查问题时非常有用：所有管理端接口走 `/admin-api/{模块}/{业务}/...`，网关按第一段路径转发到对应服务。比如访客预约的审核接口完整路径是 `PUT /admin-api/accessmgmt/visitor-appoint/audit`，看到这个 URL 就知道请求会经 Gateway 落到 accessmgmt 服务。

这一章没有贴太多业务代码，但这两千字的骨架描述，是后面八章所有内容的前置知识。下一章开始进业务，先从园区里设备最多、实时性要求最高的门禁讲起——人脸库是怎么管理的，一条通行记录又是怎么落库的。
