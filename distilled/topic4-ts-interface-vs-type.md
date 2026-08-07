# interface vs type

**它解决什么问题**
两套描述对象形状的语法长得一样，写代码每天都要选哪个，选错还会被同事/面试官质疑。

**核心机制**
- 描述对象形状时，interface 与 type 等价
- type 独占联合、元组、基本类型别名
- interface 独占声明合并：同名自动合并字段
- 扩展语法不同：interface 用 extends，type 用 &
- 二者共用同一命名空间，同标识符会冲突报 Duplicate identifier

**最容易被追问的点**
- "什么时候必须用 type？" -> 要联合/元组/基本类型别名时，interface 做不到
- "声明合并是啥？" -> 同名 interface 自动合并字段；type 同名直接报错。给第三方库补字段就靠它
- "interface 和 type 能同名吗？" -> 不能，二者共享命名空间，会报重复标识符
- "扩展子类型怎么写？" -> interface `extends`，type 用 `&` 交叉

**一句话类比**
interface 是"对象形状专用笔"只能画对象，type 是"万能别名笔"什么都能命名，但两支笔共用同一个笔筒（命名空间），插同一格会打架。
