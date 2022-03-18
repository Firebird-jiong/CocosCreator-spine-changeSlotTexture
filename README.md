# CocosCreator-spine-changeSlotTexture
前言：实现spine换装后，在网页上运行没问题，但是打包到手机上出现崩溃，换颜色没有问题。当时是两个问题：1.spine动画里有的序列帧图片是空图片，导致崩溃(不涉及换装) 2.在原生平台调用换装代码，必须要C++绑定才行

spine局部换装需要知道被换的spine插槽名，并且插槽下得有任意图片，插槽也不能隐藏，只能改透明度，否则找不到。这样就可以一套spine动画，任意的图片加载进行替换，以下是封装好的换颜色和换图片的方法：
//更改插槽颜色
changeSlotColor(sk, slotName,color = new cc.Color(255, 255,255)){
    //不能整体改变色值，目前不支持
    let slot = sk.findSlot(slotName);
    slot.color.r = color.r/255
    slot.color.g = color.g/255
    slot.color.b = color.b/255
    slot.color.a = color.a/255
},

//插槽换装
changeSlotRegion(sk, slotName, texturePath)
{
    let self = this
    cc.resources.load(texturePath, cc.Texture, (err, texture) => {
        if(err) {
            cc.log(err);
        } else {
            if(cc.sys.isNative){
                const jsbTex = new middleware.Texture2D();
                jsbTex.setPixelsHigh(texture.height);
                jsbTex.setPixelsWide(texture.width);
                jsbTex.setNativeTexture(texture.getImpl());
                // @ts-ignore
                sk.updateRegion(slotName,jsbTex);
            }else{
                self.changeSlotDetail(sk,slotName,texture)
            }
        }
    });
},

/**
 * 用外部图片局部换装
 * @param sk   骨骼动画
 * @param slotName  需要替换的插槽名称
 * @param texture   外部图片
 */
changeSlotDetail(sk, slotName, texture) {
    //获取插槽
    let slot = sk.findSlot(slotName);
    //获取挂件
    let att = slot.attachment;
    //创建region
    let skeletonTexture = new sp.SkeletonTexture();
    skeletonTexture.setRealTexture(texture)
    let page = new sp.spine.TextureAtlasPage()
    page.name = slotName
    page.uWrap = sp.spine.TextureWrap.ClampToEdge
    page.vWrap = sp.spine.TextureWrap.ClampToEdge
    page.texture = skeletonTexture
    //page.texture.setWraps(page.uWrap, page.vWrap)
    page.width = texture.width
    page.height = texture.height

    let region = new sp.spine.TextureAtlasRegion()
    region.page = page
    region.width = texture.width
    region.height = texture.height
    region.originalWidth = texture.width
    region.originalHeight = texture.height

    region.rotate = false
    region.u = 0
    region.v = 0
    region.u2 = 1
    region.v2 = 1
    region.texture = skeletonTexture
    //替换region
    att.region = region
    att.setRegion(region)
    att.updateOffset();
    
    //用于多个spine同时换装
    sk.invalidAnimationCache();
},

如果是在网页上，到这里已经结束了
-------------------------------------------------------------------------如果要打包到手机上，需要修改引擎代码：
在cocos2d-x目录下SkeletonRenderer.cpp和SkeletonCacheAnimation.cpp添加对应的方法

在SkeletonRenderer.cpp里
// 添加spine局部换肤
void SkeletonRenderer::updateRegion(const std::string& slotName,cocos2d::middleware::Texture2D *texture){
    Slot * slot = _skeleton->findSlot(slotName.c_str());
    if (slot == nullptr) {
        return;
    }
    RegionAttachment *attachment = (RegionAttachment*)slot->getAttachment();
    if (attachment == nullptr) {
         return;
    }
    Texture* texture2d = texture->getNativeTexture();
    
    float width = texture2d->getWidth();
    float height = texture2d->getHeight();
    
    float wide = texture->getPixelsWide();
    float high = texture->getPixelsHigh();
    
    
    attachment->setUVs(0, 0, 1, 1, false);
    attachment->setRegionWidth(wide);
    attachment->setRegionHeight(high);
    attachment->setRegionOriginalWidth(wide);
    attachment->setRegionOriginalHeight(high);
    attachment->setWidth(wide);
    attachment->setHeight(high);
    
    texture->setPixelsWide(width);
    texture->setPixelsHigh(height);
    texture->setRealTextureIndex(1);
    
    
    AttachmentVertices *attachV = (AttachmentVertices*) attachment->getRendererObject();
    if (attachV->_texture == texture) {
        return;
    }
    
    CC_SAFE_RELEASE(attachV->_texture);
    attachV->_texture = texture;
    CC_SAFE_RETAIN(texture);
    
    V2F_T2F_C4B *vertices = attachV->_triangles->verts;
    for (int i = 0, ii = 0; i < 4; ++i, ii += 2) {
        vertices[i].texCoord.u = attachment->getUVs()[ii];
        vertices[i].texCoord.v = attachment->getUVs()[ii + 1];
    }
    attachment->updateOffset();
    slot->setAttachment(attachment);
}

在SkeletonCacheAnimation.cpp里
// 添加spine局部换装
void SkeletonCacheAnimation::updateRegion(const std::string& slotName, cocos2d::middleware::Texture2D* texture){
    _skeletonCache->updateRegion(slotName,texture);
}

添加完以上2个脚本后，注意把对应的.h头文件也改下，然后就可以运行cocos2dx/tools/tojs/genbindings.py进行jsb自动绑定，绑定是否成功可以看Creator\2.4.4\resources\cocos2d-x\cocos\scripting\js-bindings\auto\jsb_cocos2dx_spine_auto.cpp文件夹下是否有相应的方法。

绑定成功后我们需要修改jsb adapter以提供给js层调用，
adapter在引擎安装目录下/Resources/builtin/jsb-adapter/engine/jsb-spine-skeleton.js ，添加如下方法：
    // 添加spine局部换装
    skeleton.updateRegion = function(slotsName, jsbTex2d){
        if(this._nativeSkeleton){
            this._nativeSkeleton.updateRegion(slotsName, jsbTex2d);
            return true;
        }
        return false;
    }


最后导出安卓工程后把修改的c++代码和绑定生成的代码进行替换就行了。关于jsb自动绑定需要配置安卓NDK_ROOT、python的环境变量PYTHON_BIN、还要下用到的库pyyaml和Cheetah-2.4.4.tar.gz,如果cmd运行提示找不到库，看看genbindings.py里相关引用是否有
如果不想自己修改c++代码和最后jsb的绑定，直接下载release里面省事版安装包
