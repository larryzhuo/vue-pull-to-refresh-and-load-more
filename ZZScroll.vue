<template>
  <div :id="wrapperId" class="wrapper">
  	<div :id="scrollId" class="scroll">
      <div v-if="needRefresh" class="pullup">
        <div v-show="pullupStatus==1">↓下拉刷新</div>
        <div v-show="pullupStatus==2">↑释放立即刷新</div>
        <div v-show="pullupStatus==3">加载中...</div>
        <div v-show="pullupStatus==4">刷新成功</div>
        <div v-show="pullupStatus==5">刷新失败，请重试</div>
      </div>
      <slot></slot>
      <div v-if="needRefresh && hasMoreData" class="pulldown">
        <!-- <div v-show="pulldownStatus==1">上拉加载更多</div> -->
        <div v-show="pulldownStatus==2">加载中...</div>
        <div v-show="pulldownStatus==4">加载失败，请重试</div>
      </div>
      <div v-if="!hasMoreData && moreTip" class="noData">没有更多数据了</div>
    </div>
  </div>
</template>

<script>
// import IScroll from './iscroll-probe.js'
import IScroll from './iscroll-lite-zly.js'


const pullupH = 40;
const pullDownH = 40;

export default {
  components: {IScroll},

  props: {
    needRefresh: {
      type: Boolean,
      default: false
    },
    // hasMoreData: {
      // type: Boolean,
      // default: true
    // },
    refreshCall: {
      type: Function,
      default: function(){}
    },
    loadMoreCall: {
      type: Function,
      default: function(){}
    },
    scrollX: {
      type: Boolean,
      default: false
    },
    stylesObj: {
      type: Object,
      default: function(){
        return {};
      }
    },
    wrapperId: {
      type: String,
      default: 'wrapper'
    },
    scrollId: {
      type: String,
      default: 'scroll'
    },
    initLoadMore: {
      type: Boolean,
      default: false
    },
    hasMoreData: {
      type: Boolean,
      default: true
    },
    moreTip: {      //是否需要这个提示
      type: Boolean,
      default: true
    },
    hswipeStopScroll: {     //scrollY时，横向滑动禁止纵向滑动
      type: Boolean,
      default: false
    }
  },

  data () {
    return {
      myScroll: undefined,
      pullupStatus: 1,     /*下拉状态包含1，2, 3, 4*/
      isTouch: false,
      pulldownStatus: 2,
      isLoading: false,
      moreCallTimer: undefined,  /* 加载更多改标志位还是会导致执行多次，scroll事件调用太快，采用setTimeout处理 */

      isTextArea: false
      // refTimer: undefined
    }
  },


  ready () {
    //先禁用原生滚动（除开textarea输入框），避免微信整个窗口滑动
    let self = this;
    document.addEventListener('touchstart', function(e){
      var tag = e.target.tagName.toLowerCase();
      if(tag!='input' && tag!='textarea'){                        //处理iscroll页面中input和textarea点击其他位置失去焦点
        self.inputBlur(document.querySelectorAll('input'));
        self.inputBlur(document.querySelectorAll('textarea'));
        self.isTextArea = false;
      } else {
        self.isTextArea = true;
      }
    }, false);
    document.addEventListener('touchmove', function (e) {
      if(!self.isTextArea){                     //处理textarea中文字超出可滑动
        e.preventDefault();
      } else {
        e.stopImmediatePropagation()
      }
    }, false);


    if(!this.scrollX) {
      if(this.needRefresh) {
        this.myScroll = new IScroll('#'+this.wrapperId, {
          probeType: 3,
          touchstartCall: this.tsCall,
          touchendCall: this.teCall,
          pullupH: pullupH,
          hswipeStopScroll: this.hswipeStopScroll,
          scrollY: true,
          scrollX:false,
          click: true
        });
        this.myScroll.on('scroll', function(){
            if(self.myScroll.y>0){
              self.pullupFunc();
            } else if( self.showLoadMore() ){    //激发下拉加载更多
              self.pulldownFunc();
            }
        });
        if(this.initLoadMore){
          self.pulldownFunc();
        }
      } else {
        this.myScroll = new IScroll('#'+this.wrapperId, {
          scrollY: true,
          scrollX:false,
          disableMouse: true,
          disablePointer: true,
          hswipeStopScroll: this.hswipeStopScroll,
          click: true
        });
      }
    } else {
      this.myScroll = new IScroll('#'+this.wrapperId, {     //横向滚动
        scrollX: true,
        scrollY: false,
        styleObj: this.stylesObj,
        click: true
      });
    }
  },

  methods: {
    inputBlur (inputs) {
      Array.prototype.forEach.call(inputs, function(v, i){
        v.blur();
      });
    },
    tsCall () {
      this.isTouch = true;
      // clearTimeout(this.refTimer);
      // console.log('timeout cancel' + this.refTimer);
    },
    teCall () {
      this.isTouch = false;
      if(this.pullupStatus == 2){   /*激发下拉操作了*/
        this.pullupStatus = 3;
        this.myScroll.setRefresh(true);       //teCall调用之后调用resetPosition 回滚到pullupH高度
        this.refreshCall(this.scrollId);
      }
    },
    pullupFunc () {
      if(this.isTouch && this.pullupStatus!=3){           /*刷新一定是手动touch, pullupStatus3去除正在加载中的情况*/
        let y = this.myScroll.y;
        if(y > 0 && y < pullupH * 2){
          this.pullupStatus = 1;
        } else if(y >= pullupH * 2){
          this.pullupStatus = 2;
        }
      }
    },
    pulldownFunc () {
      if(this.hasMoreData && !this.isLoading){
        this.isLoading = true;
        this.pulldownStatus = 2;
        this.loadMoreCall(this.scrollId);
      }
    },
    showLoadMore () {               //判断是否达到下拉加载更多的时机(maxScrollY==0时不足一页)
      return (this.myScroll.y<this.myScroll.maxScrollY+pullupH/2) || (this.myScroll.maxScrollY==0);
    },
  },


  events: {
    //监听父组件$broadcast msg
    "refresh-finish": function(msg){
      let self = this;
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';
      if(self.scrollId == msg.scrollId){    //区分一个页面同时有多个scroll
        if(msg.errorCode == 0)
          self.pullupStatus = 4;
        else
          self.pullupStatus = 5;
        self.myScroll.setRefresh(false);
        if(self.hasMoreData)
          self.pulldownStatus = 2;     //重新修改load-finish状态。hasMoreData的双向绑定没有这么快生效，要放在setTimeout中
      }
    },

    "load-finish": function(msg){
      let self = this;
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';
      if(self.scrollId == msg.scrollId){    //区分一个页面同时有多个scroll
        console.log('hasmore in loadfinish:'+self.hasMoreData);
        if(self.hasMoreData){
          setTimeout(function(){
            self.myScroll.refresh();
            //查看是否还需要加载，可能还是不足一页
            self.isLoading = false;   //修改标志位
            if(msg.errorCode == 0)
              self.pulldownStatus = 2;
            else
              self.pulldownStatus = 4;
            console.log(self.myScroll.maxScrollY+'------showLoadMore:'+self.showLoadMore()+'---------isLoading:'+self.isLoading);
            self.$nextTick(function () {
              if(msg.errorCode==0 && self.showLoadMore()){        //返回值正确才继续检测还能不能加载更多
                self.pulldownFunc();
                return;
              }
            });
          }, 200);
        }
      }
    },

    "scroll-refresh": function(msg){
      // console.log('refresh'+this.scrollId+'---'+msg.scrollId);
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';

      if(this.scrollId == msg.scrollId){
        this.myScroll.refresh();
      }
      return true;      //不同于 DOM 事件，Vue 事件在冒泡过程中第一次触发回调之后自动停止冒泡，除非回调明确返回 true。(为了让scroll中的scroll能监听到，必须返回true)
    },

    "scroll-to-element": function(msg){
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';
      if(this.scrollId == msg.scrollId){
        this.myScroll.scrollToElement(msg.el, 200);
      }
    },

    "scroll-zero": function(msg){
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';
      if(this.scrollId == msg.scrollId){
        this.myScroll.scrollTo(0,0);
      }
    },

    "refresh": function(msg){
      msg = msg || {};
      msg.scrollId = msg.scrollId || 'scroll';
      if(this.scrollId == msg.scrollId){
        this.myScroll.refresh();
      }
    }

  }
}
</script>

<style scoped lang="less">
.pullup {
  height: 40px;
  line-height: 40px;
  position: absolute;
  width: 100%;
  text-align: center;
  top: -40px;
}
.pulldown, .noData{
  height:40px;
  line-height:40px;
  text-align: center;
}
.wrapper {
	position: absolute;
	z-index: 1;
	top: 0;
	bottom: 0;
	left: 0;
	width: 100%;
	overflow: hidden;
}

.scroller {
	position: absolute;
	z-index: 1;
	-webkit-tap-highlight-color: rgba(0,0,0,0);
	width: 100%;
	-webkit-transform: translateZ(0);
	-moz-transform: translateZ(0);
	-ms-transform: translateZ(0);
	-o-transform: translateZ(0);
	transform: translateZ(0);
	-webkit-touch-callout: none;
	-webkit-user-select: none;
	-moz-user-select: none;
	-ms-user-select: none;
	user-select: none;
	-webkit-text-size-adjust: none;
	-moz-text-size-adjust: none;
	-ms-text-size-adjust: none;
	-o-text-size-adjust: none;
	text-size-adjust: none;
}
</style>
