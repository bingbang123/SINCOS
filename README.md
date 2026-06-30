一、核心配置层（app 根目录）
app/config.py — 应用配置中心
所有可配置项集中管理，通过环境变量覆盖默认值：

配置块	说明
DB_CONFIG	MySQL 连接参数（host/port/user/password/database）
JWT_*	JWT 密钥、算法（HS256）、过期时间（24h）
UPLOAD_DIR / MAX_UPLOAD_SIZE_MB	课表文件上传目录和大小限制
LLM_*	大模型配置（默认 DeepSeek，兼容 OpenAI）
REMINDER_*	提醒规则（deadline 前 3/1 天提醒、周五催周报）
PROJECT_GROUP_CODES / ROLES / TASK_STATUSES 等	业务枚举常量（项目组代码、6 种角色、6 种任务状态、优先级、日报/周报状态等）
app/database.py — 数据库层
基于 pymysql 封装了连接管理和查询工具：

get_connection() — 创建数据库连接，使用 DictCursor 让查询结果返回 dict
get_db() — 上下文管理器，自动 commit/rollback/close
execute(sql, params) — 写操作，返回影响行数
execute_insert(sql, params) — INSERT 操作，返回自增 ID
fetch_one(sql, params) — 查单条记录
fetch_all(sql, params) — 查全部记录
fetch_page(sql, params, page, page_size) — 分页查询，返回 (rows, total)
app/schemas.py — Pydantic 数据模型
定义所有 API 的请求/响应结构，FastAPI 自动完成校验。按模块划分：

模块	关键类	字段要点
认证	LoginReq, RegisterReq, TokenResp	登录支持邮箱/学号/工号；注册含角色、学号、工号
用户管理	CreateUserReq, UpdateUserReq, AssignRoleReq	含研究方向、入学年份等字段
到场	AttendanceItem, BatchAttendanceReq	批量记录：学生ID + 出勤状态（present/absent/unknown）
课表	UpdateScheduleItemReq, ConfirmScheduleReq	课程名、教师、地点、周次、节次
日报	CreateDailyReportReq, UpdateDailyReportReq	完成工作、问题、下一步计划、工时、标签、@提及
周报	CreateWeeklyReportReq, UpdateWeeklyReportReq	更丰富：周目标、关键成果、未完成事项、原因分析、自评
任务	CreateTaskReq, UpdateTaskReq	标题、负责人、协作者、优先级、deadline、里程碑
日历	CreateEventReq, UpdateEventReq	事件类型、关联实体（任务/日报）、全天/重复规则、可见性
项目组	CreateProjectReq, UpdateProjectReq, AddMemberReq	项目代码、负责人（导师/PHD）、成员角色
AI	ChatReq, ProjectSummaryReq	对话消息 + 对话ID、项目组汇总请求
app/auth.py — 认证与授权
安全和权限控制的核心：

函数	作用
hash_password() / verify_password()	bcrypt 密码哈希和校验
create_token(user_id, role)	签发 JWT，payload 含 user_id、role、exp（24h）
decode_token(token)	验证 JWT，过期/无效抛出 401
get_current_user()	FastAPI Depends 依赖：从 Authorization Header 提取 token → 解码 → 查库 → 返回当前用户信息，禁用用户返回 403
require_roles(*roles)	角色守卫工厂：返回一个 Depends 依赖，校验当前用户角色是否在允许列表中
get_user_project_ids(user_id, role)	数据权限：根据角色返回能看到的项目组 ID 列表。teacher/admin 看全部，phd 看自己负责的，学生只看自己所属的
二、API 路由层（app/api/）
每个文件定义一个 APIRouter，挂载到不同的 URL 前缀。路由层只做参数校验和权限声明，核心逻辑委托给 Service 层。

app/api/auth.py — 认证路由 prefix="/api/auth"
端点	方法	功能
/login	POST	邮箱/学号/工号 + 密码登录，返回 JWT token + 用户信息
/register	POST	注册新用户（默认 role=student），自动返回 token
/logout	POST	退出登录（无状态 JWT，仅标记）
/me	GET	获取当前登录用户信息
app/api/admin.py — 管理员路由 prefix="/api/admin"
端点	方法	权限	功能
/users	GET	admin+	分页列出用户，支持按角色/状态/关键词筛选
/users	POST	admin+	管理员创建用户
/users/{id}	PUT	admin+	更新用户信息（动态 set，只更新非 null 字段）
/roles	GET	admin+	列出所有角色
/users/{id}/roles	POST	super_admin	分配角色（支持作用域 scope_type/scope_id）
/audit-logs	GET	admin+	分页查看审计日志
app/api/attendance.py — 到场记录 prefix="/api/attendance"
端点	方法	权限	功能
/daily-records	POST	科研助理/管理员	批量记录到场（ON DUPLICATE KEY UPDATE 防重复）
/daily-records/{id}	PUT	科研助理/管理员	修改单条记录
/me	GET	所有登录用户	查看自己的到场记录，支持按月筛选
(列表)	GET	非学生	分页查询所有到场记录
/statistics	GET	非学生	统计当天出勤人数 + 出勤但未交日报的学生
app/api/schedule.py — 课表路由 prefix="/api/schedules"
端点	方法	权限	功能
/upload	POST	student	上传 Word 课表文件（.doc/.docx）
/{id}/parse	POST	科研助理/管理员	解析课表：从 Word 表格提取课程信息到数据库
/me	GET	所有用户	查看自己的课表
/{id}	PUT	student	修改单条课表项
/{id}/confirm	POST	student	确认课表（状态 → manually_fixed）
(列表)	GET	非学生	查看所有学生课表
/free-busy	GET	非学生	空闲/忙碌查询：查看某天某项目组所有学生的课程安排
/upload-status	GET	科研助理/管理员	查看所有学生的课表上传状态
app/api/daily_report.py — 日报路由 prefix="/api/daily-reports"
端点	方法	权限	功能
(创建)	POST	student	创建日报草稿（自动关联当天到场记录）
/{id}	PUT	student	编辑日报（仅限 draft/need_followup 状态）
/{id}/submit	POST	student	提交日报（status → submitted）
(列表)	GET	所有用户	分页查询，支持多维度筛选（学生/项目组/日期/状态/有问题/关键词）
/{id}	GET	所有用户	查看日报详情（含附件和评论）
/{id}/comments	POST	PHD/导师/管理员	添加评论（支持 comment/request_change/approve/mark_risk 动作）
/{id}/versions	GET	所有用户	查看日报的历史版本
app/api/weekly_report.py — 周报路由 prefix="/api/weekly-reports"
端点	方法	权限	功能
(创建)	POST	student	创建周报草稿（含周目标、成果、原因分析、自评等丰富字段）
/{id}	PUT	student	编辑周报
/{id}/submit	POST	student	提交周报
/generate-draft	POST	student	从日报自动生成周报草稿（聚合当周所有日报内容）
(列表)	GET	所有用户	分页查询
/{id}	GET	所有用户	查看详情（含评论和附件）
/{id}/comments	POST	PHD/导师/管理员	添加评论
/{id}/versions	GET	所有用户	查看历史版本
app/api/task.py — 任务路由 prefix="/api/tasks"
端点	方法	权限	功能
(创建)	POST	PHD/导师/管理员	创建任务（含协作者）
/{id}	PUT	所有用户	更新任务。学生只能改自己的任务状态，不能改其他字段
(列表)	GET	所有用户	分页查询，支持 overdue_only 筛选
/kanban	GET	所有用户	看板视图：返回按 6 个状态分组的任务列
/{id}	GET	所有用户	任务详情（含协作者和评论）
/{id}/comments	POST	所有用户	添加评论
/{id}/complete	POST	所有用户	快捷完成（status → done）
app/api/calendar.py — 日历路由 prefix="/api/calendar"
端点	方法	权限	功能
/events	GET	所有用户	查询事件列表，支持时间范围/类型/项目组/优先级筛选
/overview	GET	导师/管理员	一周总览：统计紧急/高优事件、deadline、会议数
/events	POST	所有用户	创建日历事件
/events/{id}	PUT	所有者/管理员	更新事件（只能编辑自己的）
/events/{id}	DELETE	所有者/管理员	删除事件（只能删除自己的）
/sync-task-deadlines	POST	导师/管理员	同步任务 deadline 到日历（将未同步的 deadline 创建为日历事件）
app/api/project.py — 项目组路由 prefix="/api/project-groups"
端点	方法	权限	功能
(列表)	GET	所有用户	列出项目组（按角色数据权限过滤）
(创建)	POST	管理员	创建项目组
/{id}	PUT	管理员	更新项目组信息
/{id}	GET	所有用户	项目组详情（含成员列表）
/{id}/summary	GET	所有用户	项目组统计：成员数、今日提交日报数、任务完成/逾期/高风险统计
/{id}/members	POST	管理员	添加成员（ON DUPLICATE KEY UPDATE）
/{id}/members/{uid}	DELETE	管理员	移除成员
app/api/ai.py — AI 助手路由 prefix="/api/ai"
端点	方法	权限	功能
/chat	POST	所有用户	AI 对话（RAG 检索 + LLM 生成），自动管理对话历史
/conversations	GET	所有用户	列出历史对话
/conversations/{id}	GET	所有用户	查看某次对话的完整消息
/weekly-draft	POST	所有用户	AI 生成周报草稿（聚合日报，调用 LLM 写作）
/project-summary	POST	所有用户	项目组汇总（统计任务+周报）
三、Service 业务逻辑层（app/services/）
app/services/ai_service.py — AI 服务（最复杂，214 行）
核心流程：RAG 检索 → 构建上下文 → 调用 LLM → 返回结果

函数	说明
chat()	主对话：关键词提取 → 多表检索（日报/周报/任务/到场/课表）→ 拼 context → 调 LLM
_retrieve()	关键词路由：根据问题关键词匹配不同数据源（"日报"→daily_reports，"任务/deadline"→tasks，"到场/签到"→attendance，"课表"→schedules）
_call_llm()	调用 OpenAI 兼容 API（DeepSeek），含 system prompt 限制模型只基于提供的数据回答
_fallback()	LLM 调用失败时的本地降级方案（按类型分组展示检索到的数据）
generate_weekly_draft_ai()	AI 周报生成（专用 prompt）
list_conversations() / get_conversation()	对话记录查询
数据权限：student 只能查自己，phd 查负责项目组，teacher/admin 查全局。

app/services/attendance_service.py — 到场服务
函数	说明
batch_record()	批量插入/更新到场记录（ON DUPLICATE KEY UPDATE）
update_record()	动态更新到场记录
get_my_records()	查询个人到场记录（支持按月筛选）
list_records()	分页查询（phd 只能看自己项目组）
statistics()	统计出勤人数 + 找出"出勤但未交日报"的学生
app/services/schedule_service.py — 课表服务
函数	说明
upload()	保存 Word 文件到磁盘，创建课表记录
parse()	核心解析器：用 python-docx 读取 Word 表格，正则提取课程名/教师/教室/周次/节次
_parse_docx()	逐行解析逻辑，含 11 个节次的时间映射（08:00–21:40）
my_schedule() / list_all()	课表查询（phd 按项目组过滤）
free_busy()	查询指定工作日所有学生的上课情况（用于排会）
upload_status()	查看所有学生的课表上传和解析状态
app/services/report_service.py — 日报/周报服务
日报部分：

函数	说明
create_daily()	创建日报（自动关联当天到场记录，防重复）
update_daily()	更新前自动保存版本快照到 report_versions 表
submit_daily()	提交日报
list_dailies()	分页查询（支持按日期范围、有问题、关键词筛选）
get_daily()	详情（含评论 + 附件）
周报部分：

函数	说明
create_weekly()	创建周报
generate_weekly_draft()	本地聚合：从当周日報拼出周报草稿（无需 AI）
list_weeklies() / get_weekly()	查询
公共部分：

函数	说明
add_comment()	添加评论，自动联动更新日报/周报状态（request_change→need_followup, approve→feedback）
get_versions()	查看历史版本
_save_version()	版本快照（编辑前调用，JSON 序列化整条记录）
app/services/task_service.py — 任务服务
函数	说明
create_task()	创建任务 + 添加协作者
update_task()	更新任务（done 时自动设 completed_at=NOW()）
list_tasks()	分页查询，按优先级+deadline 排序
kanban()	看板视图：按 6 个状态列（todo/doing/blocked/review/done/overdue）分组返回
get_task()	详情（含协作者 + 评论）
app/services/calendar_service.py — 日历服务
函数	说明
list_events()	查询事件（普通用户看不到 private 事件）
overview()	未来 7 天事件概览
create_event() / update_event() / delete_event()	CRUD
sync_deadlines()	将任务 deadline 同步到日历（跳过已有日历事件的任务）
四、中间件层（app/middleware/）
app/middleware/audit.py — 审计中间件
拦截所有非 GET/OPTIONS/HEAD 请求：

从 Authorization header 解码 JWT 获取 user_id
记录到 audit_logs 表：谁 + 什么操作（POST/PUT/DELETE + URL路径）+ 目标模块 + 响应状态码 + IP + User-Agent
即使记录失败也不影响正常请求（try-except pass）
⚠️ 注意：此中间件在 main.py 中未被注册（main.py 只加了 CORS），可能是预留功能
五、文件间调用关系图

main.py (入口，注册路由)
  ├── app/api/auth.py ──────────→ app/auth.py (JWT/bcrypt) → app/database.py → MySQL
  ├── app/api/admin.py ─────────→ app/auth.py + app/database.py
  ├── app/api/attendance.py ────→ app/services/attendance_service.py → database.py
  ├── app/api/schedule.py ──────→ app/services/schedule_service.py → database.py
  ├── app/api/daily_report.py ──→ app/services/report_service.py → database.py
  ├── app/api/weekly_report.py ─→ app/services/report_service.py → database.py
  ├── app/api/task.py ──────────→ app/services/task_service.py → database.py
  ├── app/api/calendar.py ──────→ app/services/calendar_service.py → database.py
  ├── app/api/project.py ───────→ app/database.py (直接操作)
  └── app/api/ai.py ────────────→ app/services/ai_service.py → OpenAI API + database.py

所有 API 都依赖:
  ├── app/schemas.py (请求/响应校验)
  └── app/auth.py (认证/授权)# SINCOS
