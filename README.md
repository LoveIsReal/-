### 一个模仿18年蚂蚁森林公益账单效果的h5
![演示图](https://note.youdao.com/yws/api/personal/file/1F8BF048096C4669BA1C768BB16966A0?method=download&shareKey=f7b2c4bdb6af151d86ca585662df041d)

模仿的不是页面内容，而是动画的操作和效果。

比如：
#### 1.多个动画的衔接
这里我们需要解决2个问题：

1，手指不可能一直滑动，总会touchend，如何在下一次touchstart后，保持当前的动画状态继续执行接下来的动画
2，如何知道接下来的move中，应该执行下一个动画了？

解决方案就是设定一个全局var distance，记录当前手指的滑动距离，注意哦，这个距离是每次move结束累加的，

那么每段动画必定处于一段规定的位移中
![image](https://note.youdao.com/yws/api/personal/file/CA578BA5EE144D2ABAA2FB978511E929?method=download&shareKey=8979893779293c605b5ee3630492c66d)
这些MaxDistance就是我们提前设定好的，每段动画对应一段距离，这样就可以通过距离判断当前应该执行哪一个动画，从而实现动画的衔接。


#### 2.手指快速滑动一次的处理
我们在创作中发现如果手指快速滑动，那么一次move的距离会特别大，可能直接从一个动画直接跑到另一个动画的距离中，但是很遗憾，我们无法处理溢出的距离，即我们只将一个动画完成，而溢出的距离不去计算下一个动画，因为很难。

但是我们需要注意的是，既然我们不处理当前move溢出的距离，那么move结束我们需要累加当前滑动距离啊，这可咋整，当前move只执行了当前未执行完的动画，不用怕，下次move我们这点多出来的距离直接作为下一个动画的初始状态，谁让我们是距离驱动的动画状态呢，而且在手指快速滑动的时候看起来也不是那么的突兀，因为很快


**但是呢**，下面用到了Animation ，Animation的距离累加很奇葩，要么0 ，要么1，没有0 到1这段的距离，可以理解为只有两个点，所以溢出的距离就完蛋了，直接让动画完蛋。
怎么办呢？   很简单，我们让动画的0 1 距离大点。  
比如
envelopeAnimation_3_MaxDistance  所需的距离设定为100
这样可以包括单次move事件最大的溢出距离，就是不管你在这个动画来临之前多用力滑动，都会让动画顺利过渡


#### 3.动画根据手势正向和反向的执行
这可以算是一种特殊情况, 因为一般的动画过程都是根据当前distance计算出一个值，用这个值更新UI，所以这个值可以根据手指来回滑动 变大变小，形成正向 反向的效果，但是动画不行，动画正向反向必须定义两个动画

但是这个距离如何处理，我们是通过距离判断哪个动画该执行的，这个距离对应一个距离区间的起点和终点，而不包括区间中间的值，因为动画执行过程中，不论你怎么滑动，都不会被计入distance，通过定时器和一个boolean去禁止用户滑动距离的累加

由于上一个动画有溢出距离：所以move中这么判断是该正向还是反向

```
            else if (Math.abs(distance) < envelopeAnimation_3_MaxDistance) {
                if (!isInTimeout1) { // 表示没有正在执行信封3 start动画，那么现在可以执行正向的动画
                    drawEnvelop3Forward();
                }
            } else if (parseInt(Math.abs(distance)) == parseInt(envelopeAnimation_3_MaxDistance) && absoluteMoveDistance >= 0) {
                // 执行反向的信封3动画
                if (!isInTimeout1) {
                    drawEnvelop3Backward();
                }
            }
```
envelopeAnimation_3_MaxDistance = envelopeAnimation_2_MaxDistance + 100
规定比上一个动画的距离多100，为什么是100，看 2）  

先看一下正向的动画：

```
        function drawEnvelop3Forward() {
            isInTimeout1 = true
            // 开始我们的信旋转变大动画
            var envelop_content = document.getElementsByClassName("envelop3")[0];
            envelop_content.classList.remove("rotateAndScaleEnvelopContent_class_backward");
            envelop_content.classList.add("rotateAndScaleEnvelopContent_class_forward");
            timeout1 = setTimeout(function () {
                distance = -1 * envelopeAnimation_3_MaxDistance
                isInTimeout1 = false
            }, 1100)
        }
```
isInTimeout1 = false表示还没有执行任何这个信封Animation
当我们执行，就设置为true ，同时启动定时器，定时器结束将isIntimeout设置为false，表示可以进行接下来的手势操作，但是如果isIntimeout = true，nothing should be done
当然定时结束时间要稍微大于动画时间，定时结束后要设定距离，直接设定为最大距离

反向的定时有点不同：

```
        function drawEnvelop3Backward() {
            var envelop_content = document.getElementsByClassName("envelop3")[0];


            // 清空下一个动画reading的文字，且恢复信内容的模糊
            document.getElementsByClassName("readtxt")[0].innerHTML = ""
            if (!envelop_content.classList.contains("undofade")) {
                envelop_content.classList.remove("fade");
                envelop_content.classList.add("undofade");
            }
            //------------------------

            isInTimeout1 = true
            // 开始反向执行我们的信旋转变大动画
            envelop_content.classList.remove("rotateAndScaleEnvelopContent_class_forward");
            envelop_content.classList.add("rotateAndScaleEnvelopContent_class_backward");
            timeout1 = setTimeout(function () {
                isInTimeout1 = false
                distance = -1 * (envelopeAnimation_2_MaxDistance - 1)
            }, 1100)
        }
```
定时结束是等于上一个动画的最大距离 -1，这个 -1 是为了配合上面move的那个if条件


#### 4.惯性滑动
具体做法就是touchmove记录最新的位置和duration，且在touchend中启用interval 递减的累加distance，且调用touchmove的内部过程。
（具体实现看代码）
