# Vim的配置

```
set nu "显示行号"
set cursorline "高亮显示当前行"
syntax enable "自动开启语法高亮"
colorscheme desert "设置主题颜色"
set tabstop=4 "设置缩进为4个空格"
set showmatch "高亮显示匹配的括号"
set autoindent "自动缩进开启"
set shiftwidth=4 "自动缩进4个空格"
inoremap ( ()<ESC>i  "设置（自动补全"
inoremap [ []<ESC>i  "设置[自动补全"
inoremap { {}<ESC>i  "设置{自动补全"
inoremap ' ''<ESC>i  "设置'自动补全"
inoremap " ""<ESC>i  "设置"自动补全"

```

