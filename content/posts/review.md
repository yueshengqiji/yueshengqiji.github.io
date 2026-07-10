---
title: "Review"
date: 2026-07-10T23:24:13+08:00
draft: false
---
我写这个游戏的过程中，犯过的错误几乎覆盖了从基础语法到设计思想的每一个层面。我把它们梳理成一份完整的错误清单，按发生阶段分类，方便 以后回顾。
--------------------------------------------------------------------------------
 第一阶段：基础语法与概念理解
错误具体表现纠正构造函数写错public void charater(...) 加了个 void，导致构造方法变成了普通方法构造方法不能有返回类型，连 void 都不能写构造方法名与类名不一致public void charater(...) 的 charater 不是类名 Character构造方法名必须和类名一模一样方法调用时缺少参数health.takeDamage() 无参调用，但方法定义需要 int 参数调用方法时必须按定义时的参数列表传入对应值main 方法缺少 public 或 static多次写成 void main(String[] args) 或 static void main必须是 public static void main(String[] args)字符串比较用 ==if (name == "sakiro")必须用 equals() 比较字符串内容数组初始化语法错误new String["Anon","Soyo"]应该用 new String[]{"Anon","Soyo"} 或直接 {"Anon","Soyo"}
--------------------------------------------------------------------------------
 第二阶段：面向对象设计与继承
错误具体表现纠正类和接口混淆class clxc implements Bang 试图用 implements 实现一个类implements 只能用于接口，类用 extends 继承重写时参数列表不一致void miko(int speend) 试图重写父类的 void miko()，但加了参数重写时参数列表必须与父类方法完全一致内部类访问权限理解错误试图在其他文件直接 new MyGO()，但 MyGO 是非静态内部类非静态内部类必须通过外部类实例创建：band.new MyGO()顶级类与文件名不一致类名 cat（小写）和文件名 Cat.java（大写）不匹配public 类名必须和文件名完全一致private 字段直接访问外部试图 health.hp 直接读取血量应该通过 getHp() 方法访问static 字段误用class Character { public static String name; } 导致所有角色共享名字实例字段不应该加 staticfinal 字段未初始化enum Skill 里的 baseDamage 和 isMagical 最初没有构造方法final 字段必须在构造方法中初始化
--------------------------------------------------------------------------------
 第三阶段：战斗逻辑与游戏规则
错误具体表现纠正防御判断顺序错误先计算了伤害并扣了血，然后才判断 if (playerSkill.equals("defense"))必须先判断防御，再计算伤害防御分支中伤害计算对象搞反玩家防御时，拿玩家技能的伤害值（0）去算敌人的防御减免应该用敌人的攻击力去减玩家的防御值敌人防御时调用了错误的 calculateActualDamage敌人防御时，调用的是 player.calculateActualDamage(...)，用的是玩家的防御值应该调用 enemy.calculateActualDamage(...)，用敌人的防御值减免玩家的攻击力敌方技能名拼写错误技能数组里写的是 "defends"，但判断条件用的是 "defense"拼写必须一致while 循环条件写反int i = 0; while (i > 4) 导致循环体一次都不执行应该是 while (i < 4)治疗被 if (damage > 0) 挡住takeDamage 里 if (damage > 0) 导致负数伤害（治疗）无效去掉 if 条件，改用 hp -= damage; if (hp > maxHp) hp = maxHp;
--------------------------------------------------------------------------------
 第四阶段：重构与枚举设计
错误具体表现纠正已经引入枚举，仍然用字符串比较playerSkill.equals("defense") 和 playerSkill.equals("magic") 残留应该用 playerSkill == Skill.DEFENSE 和 playerSkill.isMagical()getSkillDamage 方法仍然用字符串比较枚举if (skill.equals("strike")) 传入的明明是 Skill 类型删除该方法，直接用 skill.getBaseDamage()保留了无用字段和方法strikedamage、magicdamage 字段和 getSkillDamage 方法在重构后不再需要及时清理未使用的 Map 变量Map<String, Character> map = new HashMap<>(); 从未被使用删除注释掉的旧代码没删防御分支注释块残留删除
--------------------------------------------------------------------------------
 第五阶段：工具链与环境配置
错误具体表现纠正IDEA 里点“构建”而不是“构建工件”导致没有生成 JAR 文件必须点 Build → Build Artifacts → BuildPython 版本兼容性问题用 Python 3.14 安装 RVC 依赖时 numba 不支持降级到 Python 3.10 并创建虚拟环境网络超时导致 pip install 失败Read timed out换国内镜像源或开启 VPNjpackage 找不到环境变量没配好使用完整路径 C:\Users\as233\.jdks\openjdk-26.0.1\bin\jpackage.exe
--------------------------------------------------------------------------------
 从这些错误里收获的核心能力
1.调试能力： 从“防御变成攻击”、“敌人从未防御”这种逻辑错误里学会了逐行推演代码执行顺序。
2.类型安全意识： 从无数个 equals 拼写错误里理解了枚举和字符串的本质区别。
3.设计思维： 从“一个文件写所有类”进化到“用枚举集中管理技能和角色”。
4.工具链经验： 从 pip install 超时、Python 版本冲突、JAR 构建失败里积累了真实项目部署经验。
5.代码清理习惯： 学会了重构后删除无用字段、方法和注释。
