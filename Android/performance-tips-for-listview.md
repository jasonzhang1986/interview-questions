* 重用 convertView
* 使用 ViewHolder (静态内部类)
* itemView 中有图片时异步加载，加载过的图片有缓存(LRUCache)
* 快速滑动时不加载图片(或者只从 MemoryCache 中加载)
* 数据分页

其他的讨论可以参考 [Link](https://github.com/android-cn/android-discuss/issues/9)
