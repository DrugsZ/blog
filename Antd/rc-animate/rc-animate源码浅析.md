### rc-animate源码浅析

#### 前言

`antd`组件库大量使用了`react-component`的组件,而`antd`我们可以理解为是对`react-component`的上层封装,比如`Form`,同时有大量的`react-component`组件并不是像`Form`一样被封装一下使用,而是在其中起到了重要的协助作用,比如负责动画效果的`rc-animate`组件,

#### 责任划分

	简单的来看一下整体组件的架构，分工明显

1. Animate组件负责统筹规划，所有子节点中每个节点分别应该应用何种效果，推入相应队列进行处理
2. `AnimateChild`组件则负责对具体要执行效果的节点进行处理，包括对`css-animate`的调用，回调函数的封装处理实际上回调函数是在`Animate`中封装，`AnimateChild`更适合比作`Animate`与`css-animate`中的润滑剂
3. `css-animate`则负责具体元素的动画执行，随后调用各种传入的回调，并不关心上层，只关心传入的这个元素

通过上面的架构，我们可以看出`rc-animate`组件的责任划分及其清晰明确，`Animate`组件作为一个容器组件，随后将更加细化的处理逻辑下放到`AnimateChild`中处理，而`Animate`则只处理整体子元素的处理，分别推入相应队列后对各个队列进行处理，我们来详细查看这三部分

#### Animate

​	我们从应用初始化的步骤来看一下整个程序的逻辑

##### 	初始化

```js
  constructor(props) {
    super(props);

    this.currentlyAnimatingKeys = {};
    this.keysToEnter = [];
    this.keysToLeave = [];

    this.state = {
      children: toArrayChildren(getChildrenFromProps(props)),
    };

    this.childrenRefs = {};
  }
```

我们可以看到,`constructor`中初始化了多个属性,从名称上我们就可以看到其作用,包括正在进行动画的元素`key`,将要移入移出的队列,对子元素引用的`map`,随后将当前子元素节点缓存至`state`中,

##### render

我们继续往下看一下`render`函数

```js
render() {
    const props = this.props;
    this.nextProps = props;
    const stateChildren = this.state.children;
    let children = null;
    if (stateChildren) {
      children = stateChildren.map((child) => {
        if (child === null || child === undefined) {
          return child;
        }
        if (!child.key) {
          throw new Error('must set key for <rc-animate> children');
        }
        return (
          <AnimateChild
            key={child.key}
            ref={node => { this.childrenRefs[child.key] = node }}
            animation={props.animation}
            transitionName={props.transitionName}
            transitionEnter={props.transitionEnter}
            transitionAppear={props.transitionAppear}
            transitionLeave={props.transitionLeave}
          >
            {child}
          </AnimateChild>
        );
      });
    }
    const Component = props.component;
    if (Component) {
      let passedProps = props;
      if (typeof Component === 'string') {
        passedProps = {
          className: props.className,
          style: props.style,
          ...props.componentProps,
        };
      }
      return <Component {...passedProps}>{children}</Component>;
    }
    return children[0] || null;
  }
```

在`render`函数中,大致做了两件事,

1. 对当前所有子节点进行包装,也就是通过`AnimateChild`包装每一个子节点,然后获取其`ref`存储至我们前面所说的`childrenRefs`属性中,(`AnimateChild`我们稍后再说,目前只需要记住上面两点就可以)
2. 对所有子节点进行再包装,也就是我们要传入的`component`属性,也就是我们可以自定义容器组件,这个没什么好说的,如果没有传入则使用`span`

##### componentDidMount

`render`结束后,`componentDidMount`生命周期函数会被调起,

```js
  componentDidMount() {
    const showProp = this.props.showProp;
    let children = this.state.children;
    if (showProp) {
      children = children.filter((child) => {
        return !!child.props[showProp];
      });
    }
    children.forEach((child) => {
      if (child) {
        this.performAppear(child.key);
      }
    });
  }
```



在`componentDidMount`中我们看到其首先获取我们之前缓存的子元素节点,随后通过`showProp`属性筛选出来所有配置为显示项的子节点,推入`children`队列,随后遍历调用`performAppear`方法,可以看到`componentDidMount`生命周期函数是极其简单的,只是做了两件事,筛选和遍历,而`performAppear`我们从字面意思上来看是执行原本就存在的动画,那我们先不管他,跟随`React`的生命周期继续往下

##### componentWillReceiveProps

`componentWillReceiveProps`函数可以说是当前整个组件的核心

我们现在自己想象一下,如果我们要实现一个动画调节的容器组件,最重要也是最核心的就是我们要分辨哪些元素应该应用哪些动画,也就是说,我们需要知道哪些是移入,哪些是移除,也就是我们在初始化中提到的`keysToEnter`和`keysToLeave`两个队列.而要分辨的时机就是在`componentWillReceiveProps`生命周期中,我们可以对新旧两组子元素进行对比,这也就是`state.children`的作用,我们可以认为`state.children`是一个缓冲区,它存储了新旧子节点中所有节点,这其中包括我们提到了,没存在将要移入的,存在将要移除的,原本一直存在的,我们具体看一下代码的处理

```js
componentWillReceiveProps(nextProps) {
    this.nextProps = nextProps;
    const nextChildren = toArrayChildren(getChildrenFromProps(nextProps));
    const props = this.props;
    // exclusive needs immediate response
    if (props.exclusive) {
      Object.keys(this.currentlyAnimatingKeys).forEach((key) => {
        this.stop(key);
      });
    }
    const showProp = props.showProp;
    const currentlyAnimatingKeys = this.currentlyAnimatingKeys;
    // last props children if exclusive
    const currentChildren = props.exclusive ?
      toArrayChildren(getChildrenFromProps(props)) :
      this.state.children;
    // in case destroy in showProp mode
    let newChildren = [];
    if (showProp) {
      currentChildren.forEach((currentChild) => {
        const nextChild = currentChild && findChildInChildrenByKey(nextChildren, currentChild.key);
        let newChild;
        if ((!nextChild || !nextChild.props[showProp]) && currentChild.props[showProp]) {
          newChild = React.cloneElement(nextChild || currentChild, {
            [showProp]: true,
          });
        } else {
          newChild = nextChild;
        }
        if (newChild) {
          newChildren.push(newChild);
        }
      });
      nextChildren.forEach((nextChild) => {
        if (!nextChild || !findChildInChildrenByKey(currentChildren, nextChild.key)) {
          newChildren.push(nextChild);
        }
      });
    } else {
      newChildren = mergeChildren(
        currentChildren,
        nextChildren
      );
    }

    // need render to avoid update
    this.setState({
      children: newChildren,
    });

    nextChildren.forEach((child) => {
      const key = child && child.key;
      if (child && currentlyAnimatingKeys[key]) {
        return;
      }
      const hasPrev = child && findChildInChildrenByKey(currentChildren, key);
      if (showProp) {
        const showInNext = child.props[showProp];
        if (hasPrev) {
          const showInNow = findShownChildInChildrenByKey(currentChildren, key, showProp);
          //之前存在但是showProp为false  所以未显示,现在要显示了
          if (!showInNow && showInNext) {
            this.keysToEnter.push(key);
          }
        } else if (showInNext) {
          this.keysToEnter.push(key);
        }
      } else if (!hasPrev) {
        this.keysToEnter.push(key);
      }
    });

    currentChildren.forEach((child) => {
      const key = child && child.key;
      if (child && currentlyAnimatingKeys[key]) {
        return;
      }
      const hasNext = child && findChildInChildrenByKey(nextChildren, key);
      if (showProp) {
        const showInNow = child.props[showProp];
        if (hasNext) {
          const showInNext = findShownChildInChildrenByKey(nextChildren, key, showProp);
          if (!showInNext && showInNow) {
            this.keysToLeave.push(key);
          }
        } else if (showInNow) {
          this.keysToLeave.push(key);
        }
      } else if (!hasNext) {
        this.keysToLeave.push(key);
      }
    });
  }
```

我们来逐步分析一下

1. 首先,通过相同的方式解析新`props`中的子节点,随后判断是否传入了`exclusive`,也就是是否只允许一组动画进行,如果是,则调用下面的语句

   ```js
   Object.keys(this.currentlyAnimatingKeys).forEach((key) => {
       this.stop(key);
   });
   ```

   我们从字面意思中可以看到,对`currentlyAnimatingKeys`队列,也就是正在执行的动画队列每个元素调用停止,我们姑且这样认为,继续向下

   ```js
    const currentChildren = props.exclusive ?
         toArrayChildren(getChildrenFromProps(props)) :
         this.state.children;
   ```

   这个我们暂且不管,我们当做我们并没有传入`exclusive`,那么取值为`this.state.children;`,也就是在`constructor`中存储下来的子节点

2. ```js
   if (showProp) {
         currentChildren.forEach((currentChild) => {
           const nextChild = currentChild && findChildInChildrenByKey(nextChildren, currentChild.key);
           let newChild;
             
             //判断1
           if ((!nextChild || !nextChild.props[showProp]) && currentChild.props[showProp]) {
             newChild = React.cloneElement(nextChild || currentChild, {
               [showProp]: true,
             });
           } else {
             newChild = nextChild;
           }
           if (newChild) {
             newChildren.push(newChild);
           }
         });
       //判断2
         nextChildren.forEach((nextChild) => {
           if (!nextChild || !findChildInChildrenByKey(currentChildren, nextChild.key)) {
             newChildren.push(nextChild);
           }
         });
       } else {
         newChildren = mergeChildren(
           currentChildren,
           nextChildren
         );
       }	
   
     	// need render to avoid update
   	this.setState({
         children: newChildren,
       });
   ```

   随后进入到核心步骤,因为核心都在`showProp`为`true`的判断项,我们来看一下,我们对上面获取到的`currentChildren`进行遍历,对每一个组件根据`key`通过`findChildInChildrenByKey`函数在新`props`的子节点中进行查找,查找在新的子节点中是否还存在这个子节点,随后继续进行判断,如果新节点不再存在或者新节点的`showProp`属性为`false`,同时原先缓存子节点中存在该节点,则克隆一个`showProp`为`true`的子节点赋值给`newChild`,如果判断未通过,则直接将`nextChild`赋值给`newChild`,随后只要`newChild`存在值,则将其推入`newChildren`中,判断2则对新`props`中的所有子节点进行遍历,新节点的处理则非常简单,如果当前节点值为`false`或者是我们之前缓存的节点中没有找到新节点,则将其推入`newChildren`,

   现在我们回过头来看,判断1主要是计算了之前没有,或者之前没显示,也就是将要移入的又或者是一直存在的子节点,而判断2则计算了将要移除的子节点,随后将他们赋值到`state.children`,这也就是我们前面说到的缓存的作用,他综合了新旧两个子节点中所有要执行动画的子节点,缓存下来,等待后续的进一步处理

3. 队列处理

   ```js
   nextChildren.forEach((child) => {
         const key = child && child.key;
         if (child && currentlyAnimatingKeys[key]) {
           return;
         }
         const hasPrev = child && findChildInChildrenByKey(currentChildren, key);
         if (showProp) {
           const showInNext = child.props[showProp];
           if (hasPrev) {
             const showInNow = findShownChildInChildrenByKey(currentChildren, key, showProp);
             //之前存在但是showProp为false  所以未显示,现在要显示了
             if (!showInNow && showInNext) {
               this.keysToEnter.push(key);
             }
           } else if (showInNext) {
             this.keysToEnter.push(key);
           }
         } else if (!hasPrev) {
           this.keysToEnter.push(key);
         }
       });
   
       currentChildren.forEach((child) => {
         const key = child && child.key;
         if (child && currentlyAnimatingKeys[key]) {
           return;
         }
         const hasNext = child && findChildInChildrenByKey(nextChildren, key);
         if (showProp) {
           const showInNow = child.props[showProp];
           if (hasNext) {
             const showInNext = findShownChildInChildrenByKey(nextChildren, key, showProp);
             if (!showInNext && showInNow) {
               this.keysToLeave.push(key);
             }
           } else if (showInNow) {
             this.keysToLeave.push(key);
           }
         } else if (!hasNext) {
           this.keysToLeave.push(key);
         }
       });
   ```

   这段代码应该很好理解,主要是根据各种属性判断将其推入相应队列中,等待下一个生命周期函数进行处理

##### componentDidUpdate

当`render`结束,进入`componentDidUpdate`生命周期,这个周期中做的事情就简单多了

```js
componentDidUpdate() {
    const keysToEnter = this.keysToEnter;
    this.keysToEnter = [];
    keysToEnter.forEach(this.performEnter);
    const keysToLeave = this.keysToLeave;
    this.keysToLeave = [];
    keysToLeave.forEach(this.performLeave);
  }
```

这里我们可以看到,只是对移入移出两个队列分别调用不同的函数,

前面我们说了,`state.Children`中存储了三种类型的子元素,移入,移出,原本就存在的,那么在更新的时候我们只需要处理移入移出,那么现在当整体重新`render`结束,我们要开始应用动画,我们可以从字面意思上看出`componentDidUpdate`就是在做这个事情,我们分别看一下`performEnter`和`performLeave`做了什么

```js
performEnter = (key) => {
    // may already remove by exclusive
    if (this.childrenRefs[key]) {
      this.currentlyAnimatingKeys[key] = true;
      this.childrenRefs[key].componentWillEnter(
        this.handleDoneAdding.bind(this, key, 'enter')
      );
    }
  }

  performLeave = (key) => {
    // may already remove by exclusive
    if (this.childrenRefs[key]) {
      this.currentlyAnimatingKeys[key] = true;
      this.childrenRefs[key].componentWillLeave(this.handleDoneLeaving.bind(this, key));
    }
  }
```

我们从这可以看到,不过是根据`key`去遍历调用我们之前存储的`AnimateChild`实例的`componentWillLeave`和`componentWillEnter`方法,并传入相应的函数,从名称来看应该是动画结束的回调函数,那么我们来看看这两个函数分别做了什么

```js
 handleDoneAdding = (key, type) => {
    const props = this.props;
    delete this.currentlyAnimatingKeys[key];
    // if update on exclusive mode, skip check
    if (props.exclusive && props !== this.nextProps) {
      return;
    }
    const currentChildren = toArrayChildren(getChildrenFromProps(props));
    if (!this.isValidChildByKey(currentChildren, key)) {
      // exclusive will not need this
      this.performLeave(key);
    } else if (type === 'appear') {
      if (animUtil.allowAppearCallback(props)) {
        props.onAppear(key);
        props.onEnd(key, true);
      }
    } else if (animUtil.allowEnterCallback(props)) {
      props.onEnter(key);
      props.onEnd(key, true);
    }
  }
 
 handleDoneLeaving = (key) => {
    const props = this.props;
    delete this.currentlyAnimatingKeys[key];
    // if update on exclusive mode, skip check
    if (props.exclusive && props !== this.nextProps) {
      return;
    }
    const currentChildren = toArrayChildren(getChildrenFromProps(props));
    // in case state change is too fast
    if (this.isValidChildByKey(currentChildren, key)) {
      this.performEnter(key);
    } else {
      const end = () => {
        if (animUtil.allowLeaveCallback(props)) {
          props.onLeave(key);
          props.onEnd(key, false);
        }
      };
      if (!isSameChildren(this.state.children,
        currentChildren, props.showProp)) {
        this.setState({
          children: currentChildren,
        }, end);
      } else {
        end();
      }
    }
  }
```

我们可以看到,这两个函数大同小异,核心确实是跟我们按照名称猜测的一样是去获取传入的各种动画状态的结束回调,值得一提的是,这两个函数都会调用`this.isValidChildByKey`函数来检测当前的`props`中是否存在当前`key`的子节点,上面注释也说的很清楚是为了防止状态过快变动,我们假设一个很简单的例子就很好理解了,

如果一个子节点经历了,移入=>移出=>再移入,按照我们上面说的处理流程来说,如果数据变更过快极有可能出现上面预防的情况,也就是再移入已经生效了,移出特效才刚刚结束,移出回调被调用,这是就要做出一定的补救措施,这也就是这两个函数这么做的原因,

好我们上面说了这么多`Animate`组件,我们再来回头看看`AnimateChild`组件,看看他作为一个协调器的作用是如何工作的

#### AnimateChild

##### 自定义生命周期

在`Animate`组件中,我们介绍了,`Animate`会调用`AnimateChild`组件实例上的某些方法,他们名称类似于`React`原有的生命周期函数,所以我为了顺口叫做自定义生命周期,(不要在意),  

```js
 componentWillEnter(done) {
    if (animUtil.isEnterSupported(this.props)) {
      this.transition('enter', done);
    } else {
      done();
    }
  }

  componentWillAppear(done) {
    if (animUtil.isAppearSupported(this.props)) {
      this.transition('appear', done);
    } else {
      done();
    }
  }

  componentWillLeave(done) {
    if (animUtil.isLeaveSupported(this.props)) {
      this.transition('leave', done);
    } else {
      done();
    }
  }
```

我们看到,这三个函数其实都是一样的,都是调用了`this.transition`同时传入动画类型和回调函数,也就是我们上面说的`performEnter`等三个处理函数中传入的`handleDoneLeaving`等函数,那么我们来看看`transition`做了什么

##### transition

```js
  transition(animationType, finishCallback) {
    const node = ReactDOM.findDOMNode(this);
    const props = this.props;
    const transitionName = props.transitionName;
    const nameIsObj = typeof transitionName === 'object';
    this.stop();
    const end = () => {
      this.stopper = null;
      finishCallback();
    };
    if ((isCssAnimationSupported || !props.animation[animationType]) &&
      transitionName && props[transitionMap[animationType]]) {
      const name = nameIsObj ? transitionName[animationType] : `${transitionName}-${animationType}`;
      let activeName = `${name}-active`;
      if (nameIsObj && transitionName[`${animationType}Active`]) {
        activeName = transitionName[`${animationType}Active`];
      }
      this.stopper = cssAnimate(node, {
        name,
        active: activeName,
      }, end);
    } else {
      this.stopper = props.animation[animationType](node, end);
    }
  }

stop() {
    const stopper = this.stopper;
    if (stopper) {
      this.stopper = null;
      stopper.stop();
    }
  }
```

我们可以看到,`transition`的核心就是构建`cssAnimate`需要的参数,随后通过`CSSAnimate`去完成动画,因为整个`Animate`组件动画可以通过多种方式配置,所以`transition`做了多种判断来寻找各种状态下的css类,

通篇`Animate`组件看下来,我们可以看到一个很常见的分治的思想,通过将不同的情况规划到不同的队列,随后分别调用除了函数来处理该状态应有的动画,大大降低了整体的复杂度,如果我们没有进行合理划分整个组件的复杂度会呈指数级上升,同时也不利于维护.同时我在阅读中也学到很多,最后还是说尽信书不如无书,如有谬误之处请不吝斧正.