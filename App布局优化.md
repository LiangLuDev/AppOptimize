
### 布局分析工具
> **Layout Inspector：** 替代 Hierarchy Viewer 。

### 布局层级

> 尽量减少布局层级和复杂度


- 尽量不要嵌套使用RelativeLayout.
- 尽量不要在嵌套的LinearLayout中都使用weight属性.
- Layout的选择, 以尽量减少View树的层级为主.
- 去除不必要的父布局.
- 善用TextView的Drawable减少布局层级
- 如果布局层级超过5层, 你就需要考虑优化下布局了~

### 善用Tag
- **<include>** 使用include来重用布局.
- **<merge>** 使用<merge>来解决include或自定义组合ViewGroup导致的冗余层级问题. 减少层级.
- **<ViewStub>** 实现View的延迟加载，避免资源的浪费，减少渲染时间，在需要的时候才加载View

