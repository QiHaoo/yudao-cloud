# 01-起步：JimuReport 集成与报表开发

## 功能设计回顾

在功能设计部分，我们了解了报表模块的核心设计思想：

1. **双引擎架构**：JimuReport（积木报表）负责传统报表，GoView 负责大屏可视化
2. **Token 适配层**：实现 `JmReportTokenServiceI` 接口，将积木报表的认证与系统现有 OAuth2 体系对接
3. **权限继承**：报表模块复用系统的用户、角色、权限体系
4. **数据源共享**：报表直接使用业务系统的数据库，无需额外配置

本章将从零开始集成 JimuReport 报表引擎。

---

## 1. JimuReport 依赖引入

### 1.1 Maven 依赖配置

```xml
<!-- 文件：yudao-module-report/yudao-module-report-server/pom.xml -->
<dependencies>
    <!-- 积木报表 -->
    <dependency>
        <groupId>org.jeecgframework.jimureport</groupId>
        <artifactId>jimureport-spring-boot-starter</artifactId>
        <version>1.6.6</version>
    </dependency>

    <!-- 积木仪表盘（可选，用于大屏可视化） -->
    <dependency>
        <groupId>org.jeecgframework.jimureport</groupId>
        <artifactId>drag-spring-boot-starter</artifactId>
        <version>1.0.6</version>
    </dependency>

    <!-- 基础依赖 -->
    <dependency>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-framework-security</artifactId>
    </dependency>
    <dependency>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-framework-tenant</artifactId>
    </dependency>
</dependencies>
```

### 1.2 配置文件

```yaml
# 文件：yudao-module-report/yudao-module-report-server/src/main/resources/application.yaml
spring:
  application:
    name: report-server

server:
  port: 8085

# 积木报表配置
jmreport:
  # 数据库表前缀
  tablePrefix: jm_
  # 是否启用数据库模式
  isMode: true
  # 文件存储路径（本地模式）
  path: /data/jmreport

# 积木仪表盘配置（可选）
drag:
  # 数据库表前缀
  tablePrefix: drag_
  # 是否启用数据库模式
  isMode: true
```

---

## 2. Token 适配层实现

### 2.1 Token 服务接口

JimuReport 提供了 `JmReportTokenServiceI` 接口用于自定义认证。我们需要实现这个接口来对接系统的 OAuth2 体系：

```java
// 文件：yudao-module-report/yudao-module-report-server/src/main/java/cn/iocoder/yudao/module/report/framework/jmreport/core/service/JmReportTokenServiceImpl.java
package cn.iocoder.yudao.module.report.framework.jmreport.core.service;

import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import cn.iocoder.yudao.framework.common.biz.system.oauth2.OAuth2TokenCommonApi;
import cn.iocoder.yudao.framework.common.biz.system.oauth2.dto.OAuth2AccessTokenCheckRespDTO;
import cn.iocoder.yudao.framework.common.biz.system.permission.PermissionCommonApi;
import cn.iocoder.yudao.framework.common.exception.ServiceException;
import cn.iocoder.yudao.framework.common.util.servlet.ServletUtils;
import cn.iocoder.yudao.framework.security.config.SecurityProperties;
import cn.iocoder.yudao.framework.security.core.LoginUser;
import cn.iocoder.yudao.framework.security.core.util.SecurityFrameworkUtils;
import cn.iocoder.yudao.framework.tenant.core.context.TenantContextHolder;
import cn.iocoder.yudao.framework.web.core.util.WebFrameworkUtils;
import cn.iocoder.yudao.module.system.enums.permission.RoleCodeEnum;
import lombok.RequiredArgsConstructor;
import org.jeecg.modules.jmreport.api.JmReportTokenServiceI;
import org.springframework.http.HttpHeaders;

import javax.servlet.http.HttpServletRequest;
import java.util.Objects;

/**
 * 积木报表的 Token 服务实现
 *
 * 核心职责：
 * 1. 验证 Token 有效性
 * 2. 获取用户信息
 * 3. 获取用户权限
 * 4. 获取租户信息
 */
@RequiredArgsConstructor
public class JmReportTokenServiceImpl implements JmReportTokenServiceI {

    /**
     * 积木 token head 头
     */
    private static final String JM_TOKEN_HEADER = "X-Access-Token";

    /**
     * auth 相关格式
     */
    private static final String AUTHORIZATION_FORMAT = SecurityFrameworkUtils.AUTHORIZATION_BEARER + " %s";

    private final OAuth2TokenCommonApi oauth2TokenApi;
    private final PermissionCommonApi permissionApi;
    private final SecurityProperties securityProperties;

    /**
     * 自定义 API 数据集的 Header
     *
     * 解决积木报表调用系统 API 时的 Token 传递问题
     */
    @Override
    public HttpHeaders customApiHeader() {
        // 读取积木报表前端传递的 token
        HttpServletRequest request = ServletUtils.getRequest();
        String token = request.getHeader(JM_TOKEN_HEADER);

        // 设置到 yudao 系统的 token
        HttpHeaders headers = new HttpHeaders();
        headers.add(securityProperties.getTokenHeader(), String.format(AUTHORIZATION_FORMAT, token));
        return headers;
    }

    /**
     * 校验 Token 是否有效
     */
    @Override
    public Boolean verifyToken(String token) {
        // 优先从 Security 上下文获取
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        if (!Objects.isNull(userId)) {
            return true;
        }
        // 否则通过 token 验证
        return buildLoginUserByToken(token) != null;
    }

    /**
     * 获得用户编号
     *
     * 注意：虽然方法名是 getUsername，但实际返回用户编号
     */
    @Override
    public String getUsername(String token) {
        // 优先从 Security 上下文获取
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        if (ObjectUtil.isNotNull(userId)) {
            return String.valueOf(userId);
        }
        // 否则通过 token 获取
        LoginUser user = buildLoginUserByToken(token);
        return user == null ? null : String.valueOf(user.getId());
    }

    /**
     * 基于 token 构建登录用户
     */
    private LoginUser buildLoginUserByToken(String token) {
        if (StrUtil.isEmpty(token)) {
            return null;
        }

        // 参考 TokenAuthenticationFilter 的认证逻辑
        TenantContextHolder.setIgnore(true); // 忽略租户，保证可查询到 token 信息
        LoginUser user = null;
        try {
            OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
            if (accessToken == null) {
                return null;
            }
            user = new LoginUser()
                    .setId(accessToken.getUserId())
                    .setUserType(accessToken.getUserType())
                    .setTenantId(accessToken.getTenantId())
                    .setScopes(accessToken.getScopes());
        } catch (ServiceException ignored) {
            // 认证失败，返回 null
        }

        if (user == null) {
            return null;
        }

        // 设置到 Security 上下文
        SecurityFrameworkUtils.setLoginUser(user, WebFrameworkUtils.getRequest());

        // 设置租户上下文
        TenantContextHolder.setIgnore(false);
        TenantContextHolder.setTenantId(user.getTenantId());
        return user;
    }

    /**
     * 获取用户角色
     *
     * 适配积木报表的权限控制：
     * - 系统管理员 -> 积木报表管理员
     * - 普通用户 -> 无特殊角色
     */
    @Override
    public String[] getRoles(String token) {
        LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
        if (loginUser == null) {
            return null;
        }

        // 设置租户上下文
        TenantContextHolder.setTenantId(loginUser.getTenantId());

        // 如果是系统管理员，转换成积木报表管理员
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        return permissionApi.hasAnyRoles(userId, RoleCodeEnum.SUPER_ADMIN.getCode()).getCheckedData()
                ? new String[]{"admin"} : null;
    }

    /**
     * 获取租户编号
     */
    @Override
    public String getTenantId() {
        LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
        if (loginUser == null) {
            return null;
        }
        return StrUtil.toStringOrNull(loginUser.getTenantId());
    }

    /**
     * 获取用户权限
     *
     * 管理员拥有积木报表的全部权限
     */
    @Override
    public String[] getPermissions(String token) {
        LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
        if (loginUser == null) {
            return null;
        }

        // 设置租户上下文
        TenantContextHolder.setTenantId(loginUser.getTenantId());

        // 管理员拥有全部权限
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        if (permissionApi.hasAnyRoles(userId, RoleCodeEnum.SUPER_ADMIN.getCode()).getCheckedData()) {
            return new String[]{
                    "drag:datasource:testConnection",  // 数据库连接测试
                    "drag:datasource:saveOrUpate",     // 数据源保存
                    "drag:datasource:delete",          // 数据源删除
                    "drag:analysis:sql",               // SQL解析
                    "drag:design:getTotalData",        // 展示Online表单数据
                    "drag:dataset:save",               // 数据集保存
                    "drag:dataset:delete",             // 数据集删除
                    "onl:drag:clear:recovery",         // 清空回收站
                    "onl:drag:page:delete"             // 数据删除
            };
        }
        return null;
    }
}
```

---

## 3. Spring Boot 配置

### 3.1 积木报表配置类

```java
// 文件：yudao-module-report/yudao-module-report-server/src/main/java/cn/iocoder/yudao/module/report/framework/jmreport/config/JmReportConfiguration.java
package cn.iocoder.yudao.module.report.framework.jmreport.config;

import cn.iocoder.yudao.framework.common.biz.system.permission.PermissionCommonApi;
import cn.iocoder.yudao.framework.security.config.SecurityProperties;
import cn.iocoder.yudao.module.report.framework.jmreport.core.service.JmOnlDragExternalServiceImpl;
import cn.iocoder.yudao.module.report.framework.jmreport.core.service.JmReportTokenServiceImpl;
import cn.iocoder.yudao.framework.common.biz.system.oauth2.OAuth2TokenCommonApi;
import cn.iocoder.yudao.module.system.api.permission.PermissionApi;
import org.jeecg.modules.jmreport.api.JmReportTokenServiceI;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

/**
 * 积木报表的配置类
 *
 * 核心职责：
 * 1. 扫描积木报表的包
 * 2. 注册 Token 服务 Bean
 * 3. 注册仪表盘扩展服务 Bean
 */
@Configuration(proxyBeanMethods = false)
@ComponentScan(basePackages = "org.jeecg.modules.jmreport") // 扫描积木报表的包
public class JmReportConfiguration {

    /**
     * 注册 Token 服务
     *
     * 用于积木报表的认证和权限控制
     */
    @Bean
    public JmReportTokenServiceI jmReportTokenService(OAuth2TokenCommonApi oAuth2TokenApi,
                                                      PermissionCommonApi permissionApi,
                                                      SecurityProperties securityProperties) {
        return new JmReportTokenServiceImpl(oAuth2TokenApi, permissionApi, securityProperties);
    }

    /**
     * 注册仪表盘扩展服务
     *
     * 提供字典、日志等扩展功能
     */
    @Bean
    @Primary
    public JmOnlDragExternalServiceImpl jmOnlDragExternalService2() {
        return new JmOnlDragExternalServiceImpl();
    }
}
```

### 3.2 Security 配置

放行积木报表的静态资源和 API：

```java
// 文件：yudao-module-report/yudao-module-report-server/src/main/java/cn/iocoder/yudao/module/report/framework/security/config/SecurityConfiguration.java
package cn.iocoder.yudao.module.report.framework.security.config;

import cn.iocoder.yudao.framework.security.config.AuthorizeRequestsCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AuthorizeHttpRequestsConfigurer;

/**
 * Report 模块的 Security 配置
 */
@Configuration("reportSecurityConfiguration")
public class SecurityConfiguration {

    @Bean("reportAuthorizeRequestsCustomizer")
    public AuthorizeRequestsCustomizer authorizeRequestsCustomizer() {
        return new AuthorizeRequestsCustomizer() {

            @Override
            public void customize(AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry registry) {
                // Swagger 接口文档
                registry.requestMatchers("/v3/api-docs/**").permitAll()
                        .requestMatchers("/webjars/**").permitAll()
                        .requestMatchers("/swagger-ui").permitAll()
                        .requestMatchers("/swagger-ui/**").permitAll();

                // Spring Boot Actuator 的安全配置
                registry.requestMatchers("/actuator").permitAll()
                        .requestMatchers("/actuator/**").permitAll();

                // Druid 监控
                registry.requestMatchers("/druid/**").permitAll();

                // 积木报表（核心配置）
                registry.requestMatchers("/jmreport/**").permitAll();

                // 积木仪表盘
                registry.requestMatchers("/drag/**").permitAll();
                registry.requestMatchers("/jimubi/**").permitAll();
            }
        };
    }
}
```

---

## 4. 数据源配置

### 4.1 数据库表初始化

JimuReport 需要以下数据库表来存储报表定义。执行官方提供的 SQL 脚本：

```sql
-- 积木报表核心表（根据官方文档执行）
-- 文件：JimuReport 数据库脚本

-- 报表分组表
CREATE TABLE `jm_report_group` (
  `id` varchar(36) NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `parent_id` varchar(36) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 报表信息表
CREATE TABLE `jm_report_info` (
  `id` varchar(36) NOT NULL,
  `name` varchar(200) DEFAULT NULL COMMENT '报表名称',
  `type` int DEFAULT NULL COMMENT '报表类型',
  `status` int DEFAULT NULL COMMENT '状态',
  `json` longtext COMMENT '报表JSON定义',
  `create_by` varchar(50) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_by` varchar(50) DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 数据源表
CREATE TABLE `jm_report_db` (
  `id` varchar(36) NOT NULL,
  `name` varchar(100) DEFAULT NULL COMMENT '数据源名称',
  `db_type` varchar(20) DEFAULT NULL COMMENT '数据库类型',
  `db_driver` varchar(100) DEFAULT NULL COMMENT '驱动类',
  `db_url` varchar(500) DEFAULT NULL COMMENT '连接地址',
  `db_username` varchar(100) DEFAULT NULL COMMENT '用户名',
  `db_password` varchar(200) DEFAULT NULL COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4.2 数据源配置示例

在积木报表的管理界面中配置数据源：

| 配置项 | 示例值 | 说明 |
|-------|-------|------|
| 数据源名称 | 电商业务库 | 自定义名称 |
| 数据库类型 | MySQL | 支持 MySQL、PostgreSQL、Oracle 等 |
| 驱动类 | com.mysql.cj.jdbc.Driver | MySQL 8.x 驱动 |
| 连接地址 | jdbc:mysql://localhost:3306/yudao_cloud | 业务数据库地址 |
| 用户名 | root | 数据库账号 |
| 密码 | **** | 数据库密码 |

---

## 5. 报表设计

### 5.1 访问报表设计器

启动应用后，访问积木报表设计器：

```
http://localhost:8085/jmreport/list
```

### 5.2 创建电商销售报表

#### 步骤一：新建报表

1. 点击「新建报表」
2. 选择「空白报表」
3. 输入报表名称：`电商销售日报`

#### 步骤二：配置数据集

1. 点击「数据集」->「数据库数据集」
2. 选择已配置的数据源
3. 编写 SQL 查询：

```sql
-- 电商销售日报 SQL 示例
SELECT
    DATE(o.create_time) AS order_date,
    COUNT(DISTINCT o.id) AS order_count,
    SUM(o.pay_price) AS total_amount,
    COUNT(DISTINCT o.user_id) AS user_count
FROM trade_order o
WHERE o.status >= 30  -- 已支付订单
    AND o.create_time >= #{startDate}
    AND o.create_time < #{endDate}
GROUP BY DATE(o.create_time)
ORDER BY order_date DESC
```

#### 步骤三：设计报表样式

使用拖拽式设计器：
1. 从左侧组件库拖拽「表格」到设计区
2. 绑定数据集字段
3. 配置表头、样式、格式化

#### 步骤四：预览与发布

1. 点击「预览」查看效果
2. 确认无误后点击「发布」

### 5.3 报表展示与导出

#### 访问报表

```
http://localhost:8085/jmreport/view/{报表ID}
```

#### 导出功能

JimuReport 内置支持：
- **Excel 导出**：支持 .xlsx 格式
- **PDF 导出**：支持分页、水印
- **Word 导出**：支持 .docx 格式
- **图片导出**：支持 PNG、JPG

---

## 6. 与现有系统集成

### 6.1 菜单集成

在系统管理中配置菜单，将报表嵌入后台管理界面：

```sql
-- 添加报表菜单
INSERT INTO system_menu (name, permission, type, sort, parent_id, path, icon, component, status)
VALUES ('销售报表', '', 2, 0, 0, '/report/sales', 'chart', 'https://localhost:8085/jmreport/view/sales_report_id', 0);

-- 添加报表管理菜单
INSERT INTO system_menu (name, permission, type, sort, parent_id, path, icon, component, status)
VALUES ('报表管理', 'report:config:query', 1, 0, 0, '/report', 'report', '', 0);
```

### 6.2 权限控制

通过 `JmReportTokenServiceImpl.getRoles()` 和 `getPermissions()` 方法实现权限控制：

1. **管理员**：拥有报表设计器的全部权限
2. **普通用户**：只能查看已发布的报表

### 6.3 多租户支持

JimuReport 通过 `getTenantId()` 方法获取租户编号，实现：
- 不同租户看到不同的报表
- 报表数据自动按租户过滤

---

## 7. GoView 大屏集成（可选）

### 7.1 GoView 项目管理

```java
// 文件：yudao-module-report/yudao-module-report-server/src/main/java/cn/iocoder/yudao/module/report/controller/admin/goview/GoViewProjectController.java
package cn.iocoder.yudao.module.report.controller.admin.goview;

import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.framework.common.pojo.PageParam;
import cn.iocoder.yudao.framework.common.pojo.PageResult;
import cn.iocoder.yudao.module.report.controller.admin.goview.vo.project.GoViewProjectCreateReqVO;
import cn.iocoder.yudao.module.report.controller.admin.goview.vo.project.GoViewProjectRespVO;
import cn.iocoder.yudao.module.report.controller.admin.goview.vo.project.GoViewProjectUpdateReqVO;
import cn.iocoder.yudao.module.report.convert.goview.GoViewProjectConvert;
import cn.iocoder.yudao.module.report.dal.dataobject.goview.GoViewProjectDO;
import cn.iocoder.yudao.module.report.service.goview.GoViewProjectService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import javax.annotation.Resource;
import javax.validation.Valid;

import static cn.iocoder.yudao.framework.common.pojo.CommonResult.success;
import static cn.iocoder.yudao.framework.security.core.util.SecurityFrameworkUtils.getLoginUserId;

/**
 * GoView 大屏项目管理
 */
@Tag(name = "管理后台 - GoView 项目")
@RestController
@RequestMapping("/report/go-view/project")
@Validated
public class GoViewProjectController {

    @Resource
    private GoViewProjectService goViewProjectService;

    @PostMapping("/create")
    @Operation(summary = "创建项目")
    @PreAuthorize("@ss.hasPermission('report:go-view-project:create')")
    public CommonResult<Long> createProject(@Valid @RequestBody GoViewProjectCreateReqVO createReqVO) {
        return success(goViewProjectService.createProject(createReqVO));
    }

    @PutMapping("/update")
    @Operation(summary = "更新项目")
    @PreAuthorize("@ss.hasPermission('report:go-view-project:update')")
    public CommonResult<Boolean> updateProject(@Valid @RequestBody GoViewProjectUpdateReqVO updateReqVO) {
        goViewProjectService.updateProject(updateReqVO);
        return success(true);
    }

    @DeleteMapping("/delete")
    @Operation(summary = "删除项目")
    @Parameter(name = "id", description = "编号", required = true, example = "1024")
    @PreAuthorize("@ss.hasPermission('report:go-view-project:delete')")
    public CommonResult<Boolean> deleteProject(@RequestParam("id") Long id) {
        goViewProjectService.deleteProject(id);
        return success(true);
    }

    @GetMapping("/get")
    @Operation(summary = "获得项目")
    @Parameter(name = "id", description = "编号", required = true, example = "1024")
    @PreAuthorize("@ss.hasPermission('report:go-view-project:query')")
    public CommonResult<GoViewProjectRespVO> getProject(@RequestParam("id") Long id) {
        GoViewProjectDO project = goViewProjectService.getProject(id);
        return success(GoViewProjectConvert.INSTANCE.convert(project));
    }

    @GetMapping("/my-page")
    @Operation(summary = "获得我的项目分页")
    @PreAuthorize("@ss.hasPermission('report:go-view-project:query')")
    public CommonResult<PageResult<GoViewProjectRespVO>> getMyProjectPage(@Valid PageParam pageVO) {
        PageResult<GoViewProjectDO> pageResult = goViewProjectService.getMyProjectPage(
                pageVO, getLoginUserId());
        return success(GoViewProjectConvert.INSTANCE.convertPage(pageResult));
    }
}
```

### 7.2 GoView 项目服务

```java
// 文件：yudao-module-report/yudao-module-report-server/src/main/java/cn/iocoder/yudao/module/report/service/goview/GoViewProjectServiceImpl.java
package cn.iocoder.yudao.module.report.service.goview;

import cn.iocoder.yudao.framework.common.enums.CommonStatusEnum;
import cn.iocoder.yudao.framework.common.pojo.PageParam;
import cn.iocoder.yudao.framework.common.pojo.PageResult;
import cn.iocoder.yudao.module.report.controller.admin.goview.vo.project.GoViewProjectCreateReqVO;
import cn.iocoder.yudao.module.report.controller.admin.goview.vo.project.GoViewProjectUpdateReqVO;
import cn.iocoder.yudao.module.report.convert.goview.GoViewProjectConvert;
import cn.iocoder.yudao.module.report.dal.dataobject.goview.GoViewProjectDO;
import cn.iocoder.yudao.module.report.dal.mysql.goview.GoViewProjectMapper;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

import javax.annotation.Resource;

import static cn.iocoder.yudao.framework.common.exception.util.ServiceExceptionUtil.exception;
import static cn.iocoder.yudao.module.report.enums.ErrorCodeConstants.GO_VIEW_PROJECT_NOT_EXISTS;

/**
 * GoView 项目服务实现
 */
@Service
@Validated
public class GoViewProjectServiceImpl implements GoViewProjectService {

    @Resource
    private GoViewProjectMapper goViewProjectMapper;

    @Override
    public Long createProject(GoViewProjectCreateReqVO createReqVO) {
        // 插入
        GoViewProjectDO goViewProject = GoViewProjectConvert.INSTANCE.convert(createReqVO)
                .setStatus(CommonStatusEnum.DISABLE.getStatus());
        goViewProjectMapper.insert(goViewProject);
        // 返回
        return goViewProject.getId();
    }

    @Override
    public void updateProject(GoViewProjectUpdateReqVO updateReqVO) {
        // 校验存在
        validateProjectExists(updateReqVO.getId());
        // 更新
        GoViewProjectDO updateObj = GoViewProjectConvert.INSTANCE.convert(updateReqVO);
        goViewProjectMapper.updateById(updateObj);
    }

    @Override
    public void deleteProject(Long id) {
        // 校验存在
        validateProjectExists(id);
        // 删除
        goViewProjectMapper.deleteById(id);
    }

    private void validateProjectExists(Long id) {
        if (goViewProjectMapper.selectById(id) == null) {
            throw exception(GO_VIEW_PROJECT_NOT_EXISTS);
        }
    }

    @Override
    public GoViewProjectDO getProject(Long id) {
        return goViewProjectMapper.selectById(id);
    }

    @Override
    public PageResult<GoViewProjectDO> getMyProjectPage(PageParam pageReqVO, Long userId) {
        return goViewProjectMapper.selectPage(pageReqVO, userId);
    }
}
```

---

## 总结

本章实现了报表模块的核心功能：

1. **JimuReport 集成**：引入依赖、配置数据源、实现 Token 适配
2. **认证对接**：实现 `JmReportTokenServiceI`，对接系统 OAuth2 体系
3. **权限控制**：管理员拥有设计器权限，普通用户只能查看报表
4. **多租户支持**：通过 `getTenantId()` 实现租户隔离
5. **GoView 大屏**：可选集成，支持可视化大屏设计

关键设计点：
- 通过 Token 适配层实现与现有认证体系的无缝集成
- 复用系统的用户、角色、权限体系
- 支持多种报表导出格式
- 支持嵌入式部署，通过菜单集成到后台管理界面

下一章将介绍测试支持和源码索引。
