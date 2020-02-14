---
layout:     post
title:      "rust语言-条件编译"
subtitle:   "  "
date:       2020-01-09 
author:     "yang"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - rust
---

### rust语言条件编译



* rust中 所有的条件编译都由通过cfg配置实现，可以支持any、all、not等的任意组合和嵌套

```rust
#[cfg(any(not(unix), all(target_os="macos", target_arch = "powerpc")))]
# fn foo() {}
```

​	     `cfg `也可以用在`if-else`语句中，

```rust
if cfg!(target_os = "macos") || cfg!(target_os = "ios") {
    println!("Think Different!");
}
```

* 启用和禁用features

```rust
[features]
default = []
foo = []
```

**注意：** default这个feature在编译的时候是会自动加上的（相当于`cargo build --feature='default'`），如果在代码中不想在default feature下编译部分代码，可以使用`#[cfg(not(unix))]`





在学习开源代码过程中，发现`Carto.toml`中的一种写法

```rust
// 工程目录结构
// work space
//   |--util
//   |--foo
//	 	Cargo.toml
//   |--doo
//		Cargo.toml
//   Cargo.toml

// 在 util 下的 Cargo.toml中，
[features]
default = []
foo = ["foo"]
doo = ["doo"]

[dependencies]
foo = { path = "../foo", optional = true }  // 必须指定为 optional
foo = { path = "../doo", optional = true }  // 必须指定为 optional
```

`foo = ["foo"]`指定feature依赖的库 名称为`foo`，这时我们在`dependencies`中必须指定foo包的位置，并设置`optional = true`。如果在编译的时候指定feature为foo，则其依赖的模块 foo会编译。



```rust
// 工程根目录下的 Cargo.toml文件
[features]
default = ["home"]
home = []
util = ["mmm/util"]

// mmm 目录下的Cargo.toml
[features]
default = []
util = ["foo", "doo", "vvv/person"]
```



* default feature指定包含的 features， `util = ["mmm/util"]`表示 `util `包含 mmm 目录下定义的 `util (feature)` 

* 再看`mmm`目录下的Cargo.toml文件，`util `包含2个模块，和最后一个feature （即`vvv`目录下的person）

  

### 总结

综上，features 中的字段 可以指向另一个feature，也可以指向 该模块可能会依赖的另一个package， 正如 error 告诉我们的信息！

```rust
Feature `unstable_utils` includes `mmio` which is neither a dependency nor another feature
```





### rust编译选项



两个主要的配置：

* `dev`（执行`cargo build`）
* `release` （执行`cargo build --release`）

两个配置对应不同的opt-level

```rust
// Cargo.toml
[profile.dev]
opt-level = 0  // 编译快, 执行慢

[profile.release]
opt-level = 3  // 编译慢, 执行快
```

同时， 可以在Cargo.toml中配置opt-level

```rust
[profile.dev]
opt-level = 1
```

这样在执行`cargo build`时， opt-level为1.

































