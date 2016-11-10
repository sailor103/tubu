<template lang="pug">
section.mask2(v-cloak
  ,v-show="show"
  ,:show.sync="dataConfirmShow"
  ,:title="dataConfirmTitle"
  ,:ok-event="okEvt"
  ,:ok-text="okTxt"
  ,:no-event="noEvt"
  ,:no-text="noTxt"
  ,@touchmove.stop.prevent)

  .confirmWindow(v-cloak
    ,v-show="show"
    ,@touchmove.stop.prevent)

    .confirmTitle(v-if="hasTitle") {{ title }}
    .confirmText
      slot
    .confirmBtn
      .confirmNo(@click.stop.prevent="onNo") {{ noText }}
      .confirmYes(@click.stop.prevent="onYes") {{okText}}
</template>

<script>
  export default {
    props: {
      show: Boolean,
      title: String,
      okText: String,
      noText: String,
      okEvent: Function,
      noEvent: Function,
    },
    computed: {
      hasTitle() {
        if (this.title === '') {
          return false;
        }
        return true;
      },
    },
    methods: {
      onYes() {
        this.show = false;
        this.okEvent();
      },
      onNo() {
        this.show = false;
        this.noEvent();
      },
    },
  };
</script>

<style lang="stylus" scoped>
.mask2 
  width             100%
  height            100%
  position          fixed
  background-color  rgba(0, 0, 0, 0.5)
  left              0
  top               0
  z-index           999

.confirmWindow 
  width 280px
  border-radius 5px
  background-color #ffffff
  position fixed
  top 50%
  left 50%
  margin-left -140px
  margin-top -100px
  z-index 2

.confirmTitle 
  height         40px
  line-height    40px
  width          100%
  text-align     center
  font-size      16px
  border-bottom  1px solid rgba(186, 186, 186, 0.5)

.confirmText 
  width               100%
  font-size           16px
  color               #000000
  top                 50px
  left                0
  text-align          left
  -webkit-box-sizing  border-box
  -moz-box-sizing     border-box
  box-sizing          border-box
  padding             15px

.confirmText ul,.confirmText ol
  margin-left  20px

.confirmText ul li,.confirmText ol li
  margin-top   5px

.confirmBtn 
  position    relative 
  border-top  1px solid rgba(186, 186, 186, 0.5)
  bottom      0
  left        0
  width       100%
  height      40px

.confirmNo 
  position      absolute
  width         50%
  height        40px
  left          0
  bottom        0
  line-height   40px
  font-size     15px
  text-align    center
  color         #888888
  border-right  1px solid rgba(151, 151, 151, 0.5)

.confirmYes 
  position     absolute
  width        50%
  height       40px
  right        0
  bottom       0
  line-height  40px
  font-size    15px
  text-align   center
  color        #FF9000

</style>
