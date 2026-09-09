# 课表同步到日历的星期偏移修复

## 原因与修改

旧代码先用系统 `Calendar.current.dateInterval(of: .weekOfYear)` 找一周起点，再把课程的 `dayOfWeek % 7` 加上去。课程采用周一 = 1、周日 = 7，这个偏移只适用于星期日为起点的周；[系统日历的周起始日会随日历和地区设置变化](https://developer.apple.com/documentation/foundation/calendar/firstweekday)。当系统从周一开始时，周一课程又加了一天而落到周二，周日则被放到本周周一。

例如校历第一周为 2026-08-31 所在周、App 从周一开始时，第 2 周的周一课程应为 2026-09-07，旧算法在系统从周一开始时得到 2026-09-08。

- `ScheduleDateContext` 统一根据 App 的「校历第一周」与「周起始日」定位日期：先找第一周起点，再加 `(周次 - 1) × 7 + (课程星期 - App 周起始日 + 7) % 7`。使用日历天数加法，避免夏令时切换造成小时偏移。
- 系统日历同步、ICS 导出及周标记使用同一日期计算；ICS 导入也按 App 的周起始日还原周次。课表、小组件、快捷指令已有的日期筛选继续使用同一个上下文。
- 重同步查询改为当前学期范围，至少覆盖 30 周并在学期前后留出一周，兼容旧算法产生的错位日期和课程减少后的清理。长范围按一年分段查询。[EventKit 会将超过四年的查询截断至最初四年](https://developer.apple.com/documentation/eventkit/ekeventstore/predicateforevents(withstart:end:calendars:))，原来的 `distantPast` 到 `distantFuture` 因而无法可靠找到当前事件。
- 重同步只处理目标日历里带 EduPal URL/旧备注标记的事件；仅凭「第几周」标题判断的旧周标记，还要求目标日历由 App 创建。不会按课程名称删除其他个人事件。

## 已有错位事件如何更新

安装修复版后，保持「同步到日历」开启，在「管理课表」左滑当前课表，点「设为当前」。该操作会重新同步，清理上述范围内的旧受管事件并写入正确日期。仅安装新版本不会自动重写此前导入的事件。

手动通过 ICS 文件导入到其他日历的事件不带 App 同步标记，需要在原目标日历中处理旧导入，再使用新导出的 ICS 文件。

## 验证（2026-09-09）

- 在临时测试副本中恢复旧日期公式，周一/周日及 ICS 日期回归出现预期失败；恢复修复后的源码后通过。
- 18 项隔离测试通过：新增 9 项日历回归及已有 9 项课表选择测试。包含全部 7 × 7 种 App/系统周起始日组合、周一与周日准确日期、跨年、夏令时、ICS 往返、旧错位事件查询范围和长范围分段。
- 隔离 SwiftPM 测试使用生产的课程模型、日期上下文、节次时间表、ICS 转换器和日历同步管理器；仅 `AppSettings` 边界替换为无账号访问的测试夹具。未请求日历权限或写入真实日历。
- macOS 主应用和 Widget 编译通过。临时验证副本移除了 Watch 嵌入依赖以绕过本机缺少 Watch 资源编译运行环境的问题，仓库的项目配置未改变。
- 本机 Xcode 报告 iOS 26.5 平台未安装，尚未进行 iPhone 上 EventKit 实际写入、权限及重同步验收。

测试文件：`CCZUHelperTests/CalendarExportTests.swift`、`CCZUHelperTests/ScheduleSelectionTests.swift`。
