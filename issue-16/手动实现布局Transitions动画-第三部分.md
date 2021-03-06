# 手动实现布局Transitions动画-第三部分

> * 原文链接 : [Manual Layout Transitions – Part 3](https://blog.stylingandroid.com/manual-layout-transitions-part-3/)
* [译文出自 :  开发技术前线 www.devtf.cn](http://www.devtf.cn)
* 译者 : [Mr.Simple](https://github.com/bboyfeiyu)
* 校对者: []()  


Layout transitions are an important aspect of Material design as they help to indicate the user flow through an app, and help to tie visual components together as the user navigates. Two important tools for achieving this are Activity Transitions (which we’ll cover in the future) and Layout Transitions which we have covered on Styling Android before. However Layout Transitions are only supported in API 19 and later. 

布局切换动画在Material design中是一个重要的方面,因为它们能够指明应用的工作流程，并且能够将UI上的可视化元素绑定在一起作为用户的导航。两个重要的工具可以实现这种效果，分别为Activity转场动画和布局动画（Layout Transitions）。然后布局动画需要在API 19及其之后才支持。

Previously we created two distinct layout states and were able to toggle between them, in this article we’ll get them animating.

上一篇文章中我们创建了两个布局代表两个视图状态，我们通过setContentView来切换它们。这篇文章我们在它们切换时添加动画效果。

 
We already have the basis for the animations that we’re going to create. We have two static layout states giving us the start and end states so we just need to animate between these states. Simple!

我们已经找掌握了关于动画的相关基础知识，现在我们就要在两个布局状态切换时加入动画。


The two layouts we defined have identical Views with identical ID, only the visibility and position changes between the two states, so we just need to detect the nature of the change and apply the appropriate translation or alpha animation to each View. It is worth remembering that, because we’re are inflating a new layout, while the type and IDs of Views may be identical in both layouts, they will be represented by different View objects. Also, by the time we have transitioned to the new layout, the old one will no longer be in scope so we cannot determine the state of the controls in the old layout at that time. Therefore we need a mechanism of storing the specific View state attributes from the Views in the old layout:

我们定义的两个布局都有相同的View以及id,两个状态的切换只是会修改这些视图的可见性以及位置。因此我们仅仅需要检测这些自然变化，然后应用合适的位置变换或者alpha动画到每个视图上。值得注意的是，由于我们加载了一个新的布局，但是两个布局中的视图类型和id都是一样的，它们代表的是两个不同的视图对象。此时，我们需要切换到一个新的布局视图中，旧的布局视图就不会出现在我们的视野中，所以我们不能确定旧布局视图的控件状态。因此我们需要一种机制来存储旧布局中特定View的状态属性。

part3/ViewState.java

```java
public final class ViewState {
    private final int top;
    private final int visibility;
 
    public static ViewState ofView(View view) {
        int top = view.getTop();
        int visibility = view.getVisibility();
        return new ViewState(top, visibility);
    }
 
    private ViewState(int top, int visibility) {
        this.top = top;
        this.visibility = visibility;
    }
 
    public boolean hasMovedVertically(View view) {
        return view.getTop() != top;
    }
 
    public boolean hasAppeared(View view) {
        int newVisibility = view.getVisibility();
        return visibility != newVisibility && newVisibility == View.VISIBLE;
    }
 
    public boolean hasDisappeared(View view) {
        int newVisibility = view.getVisibility();
        return visibility != newVisibility && newVisibility != View.VISIBLE;
    }
 
    public int getY() {
        return top;
    }
}
```

This is pretty simple because we’re only interested in the vertical offset and the visibility of each View. We also have a couple of helper methods so that we can determine the delta between a new View object.

这段代码非常简单，因为我们只关心各视图的竖直偏移量和可见性。此外也就是写辅助方法以便于我们能够确定视图对象是否发生了改变。


So now we have a mechanism for storing the state of the outgoing View objects let’s take a look at how we go about doing that. We already have the mechanism in place for switching layouts which is done in the TransitionController when it calls setContentView() on our Activity. So what we need to do is, just before we call that, capture the state of the Views in the layout that is just about to be replaced. We’ll do this using a class named TransitionAnimator which will be responsible for calculating and executing the animations. So Part3TransitionController looks like this:

此时我们已经有了一套机制来存储不在可见范围的视图的状态,下面我们来看看我们如何运用的。当我们调用setContentView时，通过TransitionController我们已经有了切换布局的机制。下一步我们需要做的就是在我们切换布局之前捕获这些视图的状态，并且替换掉。我们会通过TransitionAnimator类来实现这些功能，它会计算并且执行动画。Part3TransitionController类的代码如下 : 


part3/Part3TransitionController

```java
public class Part3TransitionController extends TransitionController {
 
    Part3TransitionController(WeakReference<Activity> activityWeakReference, AnimatorBuilder animatorBuilder) {
        super(activityWeakReference, animatorBuilder);
    }
 
    public static TransitionController newInstance(Activity activity) {
        WeakReference<Activity> activityWeakReference = new WeakReference<>(activity);
        AnimatorBuilder animatorBuilder = AnimatorBuilder.newInstance(activity);
        return new Part3TransitionController(activityWeakReference, animatorBuilder);
    }
 
    @Override
    protected void enterInputMode(Activity activity) {
        createTransitionAnimator(activity);
        activity.setContentView(R.layout.activity_part2_input);
    }
 
    @Override
    protected void exitInputMode(Activity activity) {
        createTransitionAnimator(activity);
        activity.setContentView(R.layout.activity_part2);
    }
 
    private void createTransitionAnimator(Activity activity) {
        ViewGroup parent = (ViewGroup) activity.findViewById(android.R.id.content);
        View inputView = parent.findViewById(R.id.input_view);
        View inputDone = parent.findViewById(R.id.input_done);
        View translation = parent.findViewById(R.id.translation);
 
        TransitionAnimator.begin(parent, inputView, inputDone, translation);
    }
}
```

The addition here is the createTransitionAnimator() method which finds the appropriate View objects in the current layout and begins a TransitionAnimator instance. This method gets called before activity.setContentView() in both enterInputMode() and exitInputMode() methods. The important thing to remember here is that TransitionAnimator will just transition between two layout states, so requires no prior knowledge of those layouts other that details of the Views that we’re interested in animating.

这里添加了一个createTransitionAnimator函数来查找视图，并且调用了TransitionAnimator的begin函数。这个函数在enterInputMode和exitInputMode 函数中调用Activity的setContentView之前被调用。你需要注意的是TransitionAnimator只是在两个视图状态之间进行切换，因此除了那些我们感兴趣的视图之外我们不需要对这两个布局有额外的了解。

我们看看TransitionAnimator类 : 

So let’s take a look at TransitionAnimator:

```java
public final class TransitionAnimator implements ViewTreeObserver.OnPreDrawListener {
    private final ViewGroup parent;
    private final SparseArray<ViewState> startStates;
    private final AnimatorBuilder animatorBuilder;
 
    public static void begin(ViewGroup parent, View... views) {
        SparseArray<ViewState> startStates = buildViewStates(views);
        AnimatorBuilder animatorBuilder = AnimatorBuilder.newInstance(parent.getContext());
        final TransitionAnimator transitionAnimator = new TransitionAnimator(animatorBuilder, parent, startStates);
        ViewTreeObserver viewTreeObserver = parent.getViewTreeObserver();
        viewTreeObserver.addOnPreDrawListener(transitionAnimator);
    }
 
    private TransitionAnimator(AnimatorBuilder animatorBuilder, ViewGroup parent, SparseArray<ViewState> startStates) {
        this.animatorBuilder = animatorBuilder;
        this.parent = parent;
        this.startStates = startStates;
    }
 
    private static SparseArray<ViewState> buildViewStates(View... views) {
        SparseArray<ViewState> viewStates = new SparseArray<>();
        for (View view : views) {
            viewStates.put(view.getId(), ViewState.ofView(view));
        }
        return viewStates;
    }
    .
    .
    .
}
```

The static begin() method gets called by the TransitionController to kick things off.

在TransitionController类中调用TransitionAnimator的begin函数来做一些准备。


Firstly begin() calls buildViewStates() which iterates through the Views passed in an constructs ViewState objects for each which it sores in a SparseArray indexed by the relevant View IDs. It then instantiates an AnimatorBuilder object (which we’ve discussed previously), and uses this the ViewStates and the parent layout container to construct a TransitionAnimator instance.

在begin函数中首先会调用buildViewStates函数来遍历所有传递进来的视图，并且将这些视图的状态以视图id为key存储到SparseArray对象中。然后通过AnimatorBuilder对象和parent视图和存储了视图状态的SparseArray对象来创建一个TransitionAnimator实例。

Now comes the clever bit. We have captured the states of the views from the outgoing layout which hasn’t gone anywhere yet, but we now need to do something once the new layout has been created. We can’t simply inflate the layout and introspect it because the child Views will not have correct positioning until we have gone through the measurement and layout passes which happen once it gets attached to the parent container. But what we can do is register on OnPreDrawListener with the parent container. This will enable us to get a callback just before it is about to draw next time. As the TransitionController is just about to call setContentView() on the Activity, this callback will be made once the new layout has been inflated and the measurement and layout passes have completed, but before anything is drawn.

现在的代码看起来聪明一点了。在旧布局还没有从我们的视野中消失时我们捕获了它的视图状态，但是需要在新的布局创建之前我们现在需要做些其他的事情。我们不能简单的加载一个布局并且运用它，因为布局中的子视图可能还在错误的位置，直到我们经过了测量和布局两个过程之后它们才会在正确的位置。但是现在我们做的只是在parent容器中注册了一个OnPreDrawListener.这使得我们在parent下次绘制之前能够触发一个OnPreDrawListener回调。当TransitionController类中调用setContentView函数时，这个回调会在新布局被加载、测量和布局过程完成之后被调用一次，但是这个调用会执行在视图绘制之前。

TransitionAnimator implements ViewTreeObserver.OnPreDrawListener and gets registered as the OnPreDrawLister, and its onPreDraw() method will get called just before the new layout is drawn:

TransitionAnimator类实现了ViewTreeObserver.OnPreDrawListener，并且被注册为OnPreDrawLister。它的onPreDraw函数会在新布局会绘制前调用。onPreDraw函数如下 : 

part3/TransitionAnimator.java

```java
@Override
public boolean onPreDraw() {
    ViewTreeObserver viewTreeObserver = parent.getViewTreeObserver();
    viewTreeObserver.removeOnPreDrawListener(this);
    SparseArray<View> views = new SparseArray<>();
    for (int i = 0; i < startStates.size(); i++) {
        int resId = startStates.keyAt(i);
        View view = parent.findViewById(resId);
        views.put(view.getId(), view);
    }
    Animator animator = buildAnimator(views);
    animator.start();
    return false;
}
```

The first thing it does is unregister itself – we don’t want this operation to occur before every draw operation – it’s relatively heavyweight and it will kill or animation speeds if we forget to unregister. We only want it to be invoked when we replace the layout and need to begin the layout state transition animations.

在onPreDraw函数中首先将TransitionAnimator自身从ViewTreeObserver中注销，因为我们不需要在每次绘制之前都回调onPreDraw函数。如果我们忘记了注销那么它会变得相对重量级，以至于会影响动画的流畅度。我们只需要在替换布局时回调一次onPreDraw函数，然后我们会在次函数中开始执行切换动画。

The next thing it does is iterate through the SparseArray of starting ViewStates and looks up the View matching each saved state in the current layout. It then passes this array of Views in the current to buildAnimator():

下一步我们要做的是迭代前面构建的SparseArray中的ViewStates，从ViewStates中取出视图id，然后根据这个id到parent中找到对应的视图，最后将视图存储到另一个SparseArray对象中。最后将这个SparseArray对象传递给buildAnimator函数。

part3/TransitionAnimator.java

```java
private Animator buildAnimator(SparseArray<View> views) {
    AnimatorSet animatorSet = new AnimatorSet();
    List<Animator> animators = new ArrayList<>();
    for (int i = 0; i < views.size(); i++) {
        int resId = views.keyAt(i);
        ViewState startState = startStates.get(resId);
        View view = views.get(resId);
        animators.add(buildViewAnimator(view, startState));
    }
    animatorSet.playTogether(animators);
    return animatorSet;
}
```

This builds an AnimatorSet containing all of the individual View Animators which will be run in parallel. For each View buildViewAnimator() is called which constructs the appropriate Animator:

这会构建一个包含了所有独立视图Animator的集合，这些Animator会并行的执行。在构建合适的Animator时会又调用buildViewAnimator函数。

part3/TransitionAnimator.java

```java
private Animator buildViewAnimator(final View view, ViewState startState) {
    Animator animator = null;
    if (startState.hasAppeared(view)) {
        animator = animatorBuilder.buildShowAnimator(view);
    } else if (startState.hasDisappeared(view)) {
        final int visibility = view.getVisibility();
        view.setVisibility(View.VISIBLE);
        animator = animatorBuilder.buildHideAnimator(view);
        animator.addListener(
                new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationEnd(@NonNull Animator animation) {
                        super.onAnimationEnd(animation);
                        view.setVisibility(visibility);
                    }
                });
    } else if (startState.hasMovedVertically(view)) {
        int startY = startState.getY();
        int endY = view.getTop();
        animator = animatorBuilder.buildTranslationYAnimator(view, startY - endY, 0);
    }
 
    return animator;
}
```

This determines the transition type of each View by calling the helper methods on the ViewState object which we saw earlier. There are three possible transitions: An invisible View becomes visible, a visible View becomes invisible, and a View moves vertically. For each of these we construct the appropriate Animator.

在该函数中会调用ViewState中的辅助方法确定每个视图的转换的类型。这些转换类型有三种,分别为一个invisible的视图变为visible、一个visible的视图变为invisible、在y轴上移动视图。每个转换动画我们都会构建一个对应的Animator对象。

That’s it. If we run this we can see our transitions are working quite nicely:

如果此时我们运行这个示例，我们会看到很好的效果。[视频地址](https://youtu.be/CuigoE_Hjb4)。


While this works quite nicely there is one fairly major issue with it: It is fine if all of the Views in our starting layout have corresponding Views in the end layout and vice versa. But that won’t always be the case. In the next article we’ll look at how we can adapt things further to allow for this particular case.

这些代码能够很好的工作，但是有一个明显的问题它需要起始布局中的所有的视图在结束布局都有对应的视图，也就是两个布局中都含有类型和id的子view。但是这不是并不是所有的情况下都会这样。在下一篇文章中我们看看如何适配这个特定的场景。

The source code for this article is available here.

源代码在[这里](https://github.com/StylingAndroid/ManualLayoutTransitions/tree/Part3)。