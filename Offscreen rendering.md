### Offscreen rendering
* CPU: write bits in a bitmap buffer, draw with CPU, happen in app process, 不是一般说的离屏渲染
    - CoreGraphics drawing (any class prefixed with CG*)
    - drawing with CoreText
    - `drawRect`
* GPU: performed via GPU, happend in render server process
    - CALayers with a `shouldRasterize` property set to YES
        - has caching (see below)
    - CALayers using masks (`setMasksToBounds`) and dynamic shadows (`setShadow*`)
    - Group opacity (`UIViewGroupOpacity`)
    - CALayers with corner radius

    Performance penalty: context switch from on-screen drawing to off-screen drawing
 
>
> 原贴：
> https://robots.thoughtbot.com/designing-for-ios-graphics-performance
>
> 苹果员工评论：
>
> [The] advice as far as the button is good here, but I’ve got one small correction and some bonus explanation for interested readers.
> 
> I’d like to clarify a few points about offscreen drawing as described in this post. While your list of cases which might elicit offscreen drawing is accurate, there are two grossly different mechanisms being triggered by elements of this list (each with different performance characteristics), and it’s possible that a single view will require both. Those two mechanisms have very different performance considerations.
> 
> In particular, a few (implementing drawRect and doing any CoreGraphics drawing, drawing with CoreText [which is just using CoreGraphics]) are indeed “offscreen drawing,” but they’re not what we usually mean when we say that. They’re very different from the rest of the list. When you implement drawRect or draw with CoreGraphics, you’re using the CPU to draw, and that drawing will happen synchronously within your application. You’re just calling some function which writes bits in a bitmap buffer, basically.
> 
> The other forms of offscreen drawing happen on your behalf in the render server (a separate process) and are performed via the GPU (not via the CPU, as suggested in the previous paragraph). **When the OpenGL renderer goes to draw each layer, it may have to stop for some subhierarchies and composite them into a single buffer. You’d think the GPU would always be faster than the CPU at this sort of thing, but there are some tricky considerations here. It’s expensive for the GPU to switch contexts from on-screen to off-screen drawing (it must flush its pipelines and barrier), so for simple drawing operations, the setup cost may be greater than the total cost of doing the drawing in CPU via e.g. CoreGraphics would have been. So if you’re trying to deal with a complex hierarchy and are deciding whether it’s better to use –[CALayer setShouldRasterize:] or to draw a hierarchy’s contents via CG, the only way to know is to test and measure.**
> 
> You could certainly end up doing two off-screen passes if you draw via CG within your app and display that image in a layer which requires offscreen rendering. For instance, if you take a screenshot via –[CALayer renderInContext:] and then put that screenshot in a layer with a shadow.
> 
> Also: the considerations for shouldRasterize are very different from masking, shadows, edge antialiasing, and group opacity. If any of the latter are triggered, there’s no caching, and offscreen drawing will happen on every frame; rasterization does indeed require an offscreen drawing pass, but so long as the rasterized layer’s sublayers aren’t changing, that rasterization will be cached and repeated on each frame. And of course, if you’re using drawRect: or drawing yourself via CG, you’re probably caching locally. More on this in “Polishing Your Rotation Animations,” WWDC 2012.
> 
> Speaking of caching: if you’re doing a lot of this kind of drawing all over your application, you may need to implement cache-purging behavior for all these (probably large) images you’re going to have sitting around on your application’s heap. If you get a low memory warning, and some of these images are not actively being used, it may be best for you to get rid of those stretchable images you drew (and lazily regenerate them when needed). But that may end up just making things worse, so testing is required there too.
>
