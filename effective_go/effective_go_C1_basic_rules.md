# Basic Rules

## Commentary

1. 每个 package 应该有自己对应的 package comment

## Names(命名规范)

### Package names

命名原则：short, concise, evocative；全小写，单个单词；

包的 Getter 和 Setter 命名： Getter 应该用首字母大写的包名(如包为 owner, 则 Getter 命名为 Owner)；Setter 则为 Set + 首字母大写的包名(如 SetOwner)

### interface name

接口命名一般在方法名后面加上 -er 后缀，如: Reader, Writer, Formatter, CloseNotifier

### MixedCaps

驼峰命名法在 Go 语言中更加常用
