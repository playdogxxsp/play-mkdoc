# HTTP 接口开发规范

> 本规范旨在规范 http 接口的开发，方便前后端接口的交互，提高开发效率和接口的后续维护。

---

## 一、注解的使用

**文中红色标记的注解或注解属性为强制要求。**

### 1.1 ApiOperation 注解

- **【强制】** 每个 http 接口都必须使用 `@ApiOperation` 注解标记。
  - `value` 属性：简单说明接口的功能。
  - `notes` 属性：说明接口的访问权限、前端调用入口等信息。
- **【建议】**
  - 如果接口的调用处单一，可在 `value` 或 `notes` 中说明前端入口，格式：一级页面/二级页面/按钮。
  - 如果接口有多个调用处，可在 `notes` 中说明主要前端入口，其他调用处以 JavaDoc 注释形式维护在接口上方。

**示例：**
```java
@GetMapping("/listSubmitResumes")
@ApiOperation(value = "候选人管理页，查询列表", notes = "适用于阶段：简历筛选、用人部门筛选、面试；数据操作权限：数据权限内的非辅导老师职位、简历负责人、offer负责人或超管")
public Response<CandidateManagePageVo> listSubmitResumes(@ModelAttribute CandidateManageQueryCondition condition, HttpServletRequest request){
    String ldap = getUsername(request);
    return Response.success(candidateManageLogic.listSubmitResumes(condition, ldap));
}
```

---

### 1.2 多条件查询注解

- **【强制】** 多条件（条件数量≥3）查询接口，需将条件写到以 `Param` 结尾的类中，接口入参用 `@ModelAttribute` 注解标记。
- **【强制】** `Param` 类上用 `@ApiModel` 注解标记，使用对象接收参数。
- **【强制】** 类中每个查询条件用 `@ApiModelProperty` 注解标记。
  - `value` 属性：说明条件含义、取值范围、参数格式等。
  - `notes` 属性：补充后端专用信息。
  - `hidden=true`：不暴露给前端。
  - `required=true`：必传条件。

**示例：**
```java
@Data
@ApiModel("候选人管理查询条件类")
public class CandidateManageQueryParam {
    @ApiModelProperty("查询的职位")
    private List<Integer> queryPositionIds = null;
    @ApiModelProperty(value = "用户有权限的职位", hidden = true)
    private List<Integer> authorizedPositionIds = null;
    @ApiModelProperty(value = "一级渠道", required = true)
    private ChannelV2 channelV2 = null;
    @ApiModelProperty(value = "二级渠道；如果是全部，那么传空字符串")
    private String secChannel = "";
    @ApiModelProperty(value = "简历负责人, 格式：姓名ldap")
    private String uploaderLdap = "";
    @ApiModelProperty(value = "起始投递日期, 日期零点时间戳，毫秒")
    private long applyFromDate = 0L;
    @ApiModelProperty(value = "终止投递日期, 日期零点时间戳，毫秒")
    private long applyToDate = 0L;
    @ApiModelProperty(value = "候选人姓名", notes = "模糊查找，最左匹配")
    private String name = "";
    @ApiModelProperty(value = "候选人电话", notes = "完全匹配")
    private String phone = "";
    @ApiModelProperty(value = "当前阶段的状态，简历筛选：待筛选简历，用人部门筛选：待用人部门筛选，面试/待安排面试：待安排面试，面试/已安排：已安排面试，offer: 待发offer，入职/待入职：待入职，入职/已入职：已入职")
    private SubmitResumeStatus currentStatus = null;
    @ApiModelProperty(value = "候选人状态", hidden = true)
    private List<SubmitResumeStatus> statusList = null;
    @ApiModelProperty(value = "是否是超级管理员", hidden = true, notes = "超级管理员不受简历负责人或offer负责人的限制，可以查看列表数据")
    private boolean isSuperAdmin = false;
}
```

---

### 1.3 POST 提交数据

- **【强制】** 使用 POST 方法的接口提交数据时，使用对象类型，并用 `@RequestBody` 注解标记。

**示例：**
```java
@PostMapping("/createInterviewEvaluation")
@ApiOperation("填写面试反馈")
public Response createInterviewEvaluation(@RequestBody List<RoundEvaluationVo> roundEvaluationVo, HttpServletRequest request) {
    String username = getUsername(request);
    candidateManageInterviewLogic.createInterviewEvaluation(roundEvaluationVo, username);
    return Response.success();
}
```

---

### 1.4 其他注解

- **【强制】** `@Api`：修饰整个类，描述 Controller 作用。
- **【强制】** `@ApiParam`：单个参数描述。
- 其他注解：`@ApiResponse`、`@ApiResponses`、`@ApiIgnore`、`@ApiError`、`@ApiImplicitParam`、`@ApiImplicitParams`。

**示例：**
```java
@GetMapping("/generateWechatJSSDKSignature")
@ApiOperation(value = "获取微信js-sdk签名", notes = "对外接口，没有权限控制")
public Response generateWechatJSSDKSignature(@RequestParam("url") String url, @ApiParam("公众号所属部门") @RequestParam(defaultValue = "猿辅导") DepartmentEnum department){
    // 返回给 c 端用户，屏蔽一些错误
    try {
        return Response.success(wechatLogic.generateWechatJSSDKSignature(url, department));
    } catch (Throwable t) {
        logger.error("get wechet signature error:", t);
        throw new MsgRuntimeException("获取签名失败");
    }
}
```

---

## 二、接口返回值规范

- **【强制】** 所有 http 接口返回值使用带有泛型的 `Response` 类型，包含 `code`、`message` 和 `data` 部分。
  - 无返回值时，泛型用 `Void` 类型。
- **【强制】** `code` 用于区分不同异常类型。
- **【强制】** `message` 是异常时的错误描述信息。
- **【强制】** `data` 是返回的数据内容。
- **【强制】** `Response` 带泛型后，前端可通过 swagger 了解返回值格式和字段名称，避免联调时才知数据格式。

**示例：**
```json
{
    "code": 0,
    "message": "",
    "data": {
        // 具体数据
    }
}
```

---

## 三、接口 URL 和请求方法

- **【强制】** 接口 URL 统一为 RPC 风格：`/系统名/模块名/接口方法名`，模块名不包含 Controller。
- **【强制】** 同一 controller 类中不能使用方法重载，避免 URL 相同。
- **【强制】** 方法名称使用动宾短语，动词范围见项目规范。

**请求方法规范：**
- `GET`：获取数据，幂等，常用于查询。
- `POST`：提交数据，非幂等，常用于创建。
- `PUT`：上传数据，幂等，常用于更新。
- `DELETE`：删除资源，幂等。
- `HEAD`、`OPTIONS`、`PATCH`：根据实际功能和幂等性确定。

**实际用法：**
- 查询数据：`GET`
- 创建数据：`POST`
- 修改数据：`PUT`
- 逻辑删除（非幂等）：`POST`
- 删除数据（幂等）：`DELETE`

**示例：**
```java
@RestController
@RequestMapping("/candidateManage")
@Api("候选人管理")
public class CandidateManageController extends BaseController {
    @Autowired
    private CandidateManageLogic candidateManageLogic;
    @GetMapping("/listSubmitResumes")
    @ApiOperation(value = "候选人管理页，查询列表", notes = "适用于阶段：简历筛选、用人部门筛选、面试；数据操作权限：数据权限内的非辅导老师职位、简历负责人、offer负责人或超管")
    public Response<CandidateManagePageVo> listSubmitResumes(@ModelAttribute CandidateManageQueryCondition condition, HttpServletRequest request){
        String ldap = getUsername(request);
        return Response.success(candidateManageLogic.listSubmitResumes(condition, ldap));
    }
}
```

---

## 四、分页规范

- **【强制】** 分页请求继承 `ep-commons` 中的 `PageParam`。
- **【强制】** 分页返回使用 `ep-commons` 中的 `PageData<T>` 作为返回类型。

**请求示例：**
```json
{
    "condition1": "",
    "condition2": 2,
    "pageNo": 1,
    "pageSize": 20
}
```

**返回示例：**
```json
{
    "code": 0,
    "message": "",
    "data": {
        "count": 100,
        "list": [
            { "id": 1, "name": "test1" },
            { "id": 2, "name": "test2" }
        ]
    }
}
```

---

如需补充其他细节，可根据实际项目需求扩展。




 