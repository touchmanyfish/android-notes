# RecyclerView5-测量分析(TODO)


## 目前先这么理解
如果parent高宽都给exactly，那就按这个来；只要高宽中有一个不是，则需要先测量child然后再根据结果来推算自己的尺寸。
