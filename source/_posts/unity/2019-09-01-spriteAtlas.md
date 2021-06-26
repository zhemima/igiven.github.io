---
title:  "关于spriteAtlas打包那些事"
---



### UGUI的合图是在什么时候发生的？

Unity合大图的时机是根据设置来的。*Edit->ProjectSetting->Editor*

![1567211125212](../../assets/images/2019-09-01-spriteAtlas/1567211125212.png)

如上图，可以在打包的时候合，也可以编辑器运行的时候就合。Editor中合成的大图是放在缓存目录里：`Library/AtlasCache`。

- Enabled For Builds  （Unity将精灵打包到Sprite Atlas中，仅用于已发布的版本。Editor和Play模式引用原始源纹理而不是Sprite Atlas中的纹理。）。
- Always Enabled   （默认情况下启用此选项。Unity将选定的纹理打包到Sprite Atlases中，Sprite在运行时引用打包的Textures。但是，Sprites将在Editor模式期间引用原始未压缩的纹理。）。

简单的说:

- 如果想测试spritepacker是否生效以及代码相关..这时候选择Always Enabled..可以理解为生产模式
- 如果平时开发Enabled For Builds,只有打包的时候才构建..可以理解为开发模式



### Include in Build到底干了什么？

勾选了Include in Build后，图集资源会被打进App包体里（不是AssetBundle包）。如果图集是AssetBundle包管理的，最好不要勾选它，会造成资源双份。至于哪些资源会双份，需要实验下看看。



###  禁用Include in Build,使用Late Binding

1. 只要精灵打包到任何精灵图集内，但精灵图集未绑定为默认图集（例如未选中“Include in build”选项），精灵便会在场景中不可见。
2. 用户可以监听回调 SpriteAtlas.atlasRequested。
3. 此委托方法将提供一个要绑定的图集标签和一个接受 SpriteAtlas 资源的 System.Action。用户应按任意方式（脚本引用、Resources.load、资源包）加载该资源，并将该资源提供给 System.Action。



注意:

- SpriteAtlas.atlasRequested只会请求一次无论成功还是失败,所以要确保你的atlasRequested回调一定能返回正确的值
- 当使用addressable异步加载spriteatlas的时候,也会触发SpriteAtlas.atlasRequested,这个时候内存中可能会存在两份资源的引用..记得释放其中的一份





### 相关链接

- https://connect.unity.com/doc/Manual/SpriteAtlas