---
title: "如何使用中间选项简化NixOS模块配置"
date: 2026-08-18T20:21:12
tag:
  - Nix
  - NixOS
  - Linux
  - OS
  - Declarative
  - 操作系统
  - 声明式
---

## 问题

在编写NixOS模块的过程中，我们经常会遇到需要多次判断某个复杂条件（即不是简单的、由一个选项决定的条件）来决定是否做一些操作的问题。

比如假设我们有一个gui模块开关选项 `options.gui.enable`，其下有一个zen浏览器子模块开关选项 `options.gui.zen.enable`。

我们希望当用户设置 `config.gui.enable = false` 时，不管如何设置 `config.gui.zen.enable`，zen浏览器都不会被启用。

因为Nix并不存在真正的子选项，`options.gui.enable` 与 `options.gui.zen.enable` 实际上是两个平行选项，并不会相互影响。

所以如果要在配置里实现上述效果，传统的做法是用一个带 `&&` 的表达式来做判断，就像这样：

``` nix
config = lib.mkIf config.gui.enable && config.gui.zen.enable {
  # Ops...
};
```


如果zen浏览器模块是单个文件，这么做没有什么问题。

但如果zen浏览器模块的内容越来越多，以至于被拆分成了很多文件，然后每个文件里都要写这么一个判断，那就简直太地狱了。

另外一种简单的解决方式是只 `mkIf config.gui.zen.enable`，然后添加assertion，让zen启用但gui未启用时直接报错。

但是这样用户就必须额外更改配置，用户体验很不好。

## 什么是中间选项

中间选项即，我们可以创建一个选项，它只读、值由现有选项的值决定、不在文档中显示、也不能且不应被用户设置。

NixOS模块系统给予了我们创建这种选项的能力，我们只需要像如下这样使用 `mkOption`，即可创建这样的选项：

``` nix
mkOption {
  internal = true; # 内部的，即不在文档中显示
  readOnly = true; # 只读的，即不能被设置
  default = config.xxx && config.yyy; # 它的值是什么
}
```

为了更方便地创建这种选项，我们还可以创建一个工具函数：

``` nix
mkComputedOption = by: lib.mkOption {
  internal = true;
  readOnly = true;
  default = by;
}
```

当然我们也可以给这种选项设置 `type`，使其值被模块系统类型检查，这里不再赘述。

## 中间选项如何解决上面的问题

我们可以创建一个 `options.internal.final.gui.zen.enable` 的中间选项，其值为用户设置的 `config.gui.enable` 和 `config.gui.zen.enable` 的与运算：

``` nix
options.internal.final.gui.zen.enable = mkComputedOption (config.gui.enable && config.gui.zen.enable);
```


这里我们把这个中间选项放置在 `internal.final` 中，表示这是一个内部的、不应被手动设置的选项，且其是一个计算后的最终选项。

当然，命名空间不是强制的，只是一种约定，可以让我们看到选项名就知道这个选项大概是什么类型，减少误用。

之后不管zen浏览器分出了多少模块，我们都只需要 `mkIf config.internal.final.gui.zen.enable`，再也不用到处写带括号的与运算表达式了（这里顺便吐槽一下Nix的运算符优先级，写好多东西都要加括号）。

## 叠层？嵌套？

是的，中间选项时可以嵌套的，这也并不会导致无限递归，毕竟它很trivial。

比如我们可以这样：

``` nix
options.internal.gui.enable = mkComputedOption (config.gui.niri.enable || config.gui.hyprland.enable);

config = mkIf config.internal.gui.enable {
  # 任意一种桌面环境被启用都会被应用的设置
};

options.internal.final.gui.zen.enable = mkComputedOption (config.internal.gui.enable && config.gui.zen.enable);

config = mkIf config.internal.final.gui.zen.enable {
  # 任意一种桌面环境被启用，且zen被设置启用，才会被应用的设置
};
```

## 更高级（抽象）的用法——由home-manager配置决定系统配置

假设我们的模块化做得非常好，每个主机配置都可以有不同的用户，且每个用户都可以选择开关home-manager模块中的选项。

比如主机A上的用户a可以开启gui模块，但用户b则关闭gui模块。

在这种情况下，就有可能出现一台主机上没有一个用户开启gui模块的情况，而这时，再无条件把gui模块放进系统闭包就是useless的，只会增加闭包体积。

最简单的解决方式是，我们在NixOS模块中创建需要单独设置的gui模块开关，然后在home-manager模块中做assertion，使得只要有一个用户启用了gui模块但主机没有启用gui模块就报错。

但这也和文章开头的「解决方案」有相同的问题，就是增加了用户的设置复杂度。

因此，我们可以创建一个中间选项，其值在「任意一个用户启用了gui模块」时为真：

这里我们假设我们定义的NixOS模块都在os命名空间下，home-manager模块都在home命名空间下，以方便阅读，即使它们实际上并不会冲突。

``` nix
options.os.internal.gui.enable = mkComputedOption (
  config.home-manager.users # 键为用户名，值为用户home config的attrs
    |> attrValues # 我们只需要attrs的值，这会返回一个列表，每项都是一个用户的home config
    |> map (cfg: cfg.home.internal.gui.enable) # 把列表中的每个home config变换为「该用户是否启用了gui模块」
    |> any id # 若列表中任意一项为真则结果为真
);

config = lib.mkIf config.os.internal.gui.enable {
  # 需要在NixOS模块中设置的gui相关设置
};
```

这样我们就可以实现「主机上任意用户启用gui模块，则主机启用gui模块」了，对于没有任何启用gui模块的用户的主机，gui模块是关闭的，闭包可以小很多。

是的，这也并不会无限递归，lazyness，很神奇吧。

## 后记

我们通过巧妙地使用NixOS模块系统提供的 `internal` 和 `readOnly` 属性，构建了一种中间选项，其可以显著地帮助我们降低配置的复杂性。

当然，这里因为篇幅原因，使用的都是高度简化的案例，若有兴趣，可查看 [我的NixOS配置](https://codeberg.org/lialh4/nixos-config)，其中大量使用了本文讲述的中间选项，可以说是很落地实践了。
