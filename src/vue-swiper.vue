<template>
    <div class="swiper"
         :class="[direction, {'dragging': dragging}]"
         @touchstart="_onTouchStart"
         @mousedown="_onTouchStart"
         @wheel="_onWheel">
        <div class="swiper-wrap"
             v-el:swiper-wrap
             :style="{'transition':'all '+realTransitionTime+'s ease','transform' : 'translate3d(' + translateX + 'px,' + translateY + 'px, 0)'}"
             @transitionend="_onTransitionEnd">
            <div class="clone_end" v-show="loop"></div>
            <slot></slot>
            <div class="clone_first" v-show="loop"></div>
        </div>

        <div class="swiper-pagination"
             v-show="paginationVisible">
            <span class="swiper-pagination-bullet"
                  :class="{'active': $index+1===currentPage}"
                  v-for="slide in totalPage"
                  @click="paginationClickable && clickPage($index+1)" track-by="$index"></span>
        </div>
    </div>
</template>
<style lang="less" src="./vue-swiper.less"></style>
<script type="text/babel">
    const VERTICAL = 'vertical';
    const HORIZONTAL = 'horizontal';
    import {Event} from './smart.min'



    export default {
        props: {
            direction: {
                type: String,
                default: VERTICAL,
                validator: (value) => [VERTICAL, HORIZONTAL].indexOf(value) > -1
            },
            mousewheelControl: {
                type: Boolean,
                default: true
            },
            performanceMode: {
                type: Boolean,
                default: false
            },
            paginationVisible: {
                type: Boolean,
                default: false
            },
            paginationClickable: {
                type: Boolean,
                default: false
            },
            resistanceRatio: {
                default: 0.3
            },
            slides: {
                type: Array,
                default: []
            },
            autoplay: {
                type: Number,
                default: 0
            }, autoplayStopOnLast: {
                type: Boolean,
                default: false
            },
            autoplayDisableOnInteraction: {//用户操作后停止自动播放
                type: Boolean,
                default: true
            },
            loop: {
                type: Boolean,
                default: false
            },
            allowSwipeToNext: {
                type: Boolean,
                default: true
            },
            allowSwipeToPrev: {
                type: Boolean,
                default: true
            },
            onlyExternal: {//
                type: Boolean,
                default: false
            },
            page: {
                type: Number,
                default: 1
            },
            transitionTime: {
                type: Number,
                default: 0.5
            },
            skip:{
                type: Boolean,
                default: false
            }
        },
        data() {
            return {
                currentPage: 1,
                lastPage: 1,
                totalPage: 1,
                translateX: 0,
                translateY: 0,
                startTranslateX: 0,
                startTranslateY: 0,
                delta: 0,
                dragging: false,
                startPos: null,
                transitioning: false,
                slideEls: [],
                auto_play_iv: 0,
                auto_play_stoped: true,
                skiping: false
            };
        },
        ready() {
            this._onTouchMove = this._onTouchMove.bind(this);
            this._onTouchEnd = this._onTouchEnd.bind(this);
            this.slideEls = this.$els.swiperWrap.children;
            this._buildLoopDiv();
            this.totalPage = this.$els.swiperWrap.children.length - 2;
            this.startAutoPlay();
            var me = this;

            me.rm_widow_event = Event.windowEvent('resize', function () {
                me._revert();
            });
            setTimeout(this.skipTo, 0, this.page);
        },
        destroyed(){
            this.rm_widow_event();
        }
        ,
        computed: {
            realTransitionTime: function () {

                return (this.dragging || this.skiping) ? 0 : this.transitionTime;
            },
            isBeginning: function () {

                return this.currentPage == 1;
            }, isEnd: function () {
                return this.currentPage == this.totalPage;

            }
        },
        watch: {
            'slides': {
                handler: function (val, oldVal) {
                    var me = this;
                    me.slideEls = [];
                    me.slideEls = me.$els.swiperWrap.children;
                    this.totalPage = this.$els.swiperWrap.children.length - 2;
                    this._buildLoopDiv();
                }
            },
            'page': {
                handler: function (val, oldVal) {
                    if(this.skip==true){
                        this.skipTo(val);
                    }else{
                        this.setPage(val);
                    }

                }
            }
        },
        methods: {
            _buildLoopDiv(){
                if (this.loop) {
                    this.$els.swiperWrap.replaceChild(this.$els.swiperWrap.children[this.$els.swiperWrap.children.length - 2].cloneNode(true), this.$els.swiperWrap.children[0]);
                    this.$els.swiperWrap.replaceChild(this.$els.swiperWrap.children[1].cloneNode(true), this.$els.swiperWrap.children[this.$els.swiperWrap.children.length - 1]);

                } else {
                    var edom = document.createElement('div');
                    edom.style.display = 'none';
                    this.$els.swiperWrap.replaceChild(edom, this.$els.swiperWrap.children[0]);
                    this.$els.swiperWrap.replaceChild(edom.cloneNode(true), this.$els.swiperWrap.children[this.$els.swiperWrap.children.length - 1]);
                }

            },

            _fixPage(){
                if (!this.loop) {
                    return
                }
                if (this.currentPage < 1) {
                    this.skipTo(this.totalPage)
                } else if (this.currentPage > this.totalPage) {
                    this.skipTo(1)
                }
            },

            skipTo(page){
                this.skiping = true;
                this.setPage(page);
                var me = this;
                setTimeout(function () {
                    me.skiping = false;
                }, 0)


            },
            startAutoPlay(){
                if (this.autoplay > 0 && this.auto_play_stoped) {
                    this.auto_play_iv = setInterval(this.next, this.autoplay);
                    this.auto_play_stoped = false;
                }
            },
            stopAutoPlay(){
                clearInterval(this.auto_play_iv);
                this.auto_play_stoped = true;
            },
            clickPage(page){
                this.stopAutoPlay();
                this.setPage(page);
            },
            next() {
                var page = this.currentPage;
                if (!this.isEnd || this.loop) {
                    page++;
                    this.setPage(page);
                } else {
                    if (!this.dragging) {
                        this.currentPage = 1;
                    }
                    this._revert();
                }
            },
            prev() {
                var page = this.currentPage;
                if (!this.isBeginning || this.loop) {
                    page--;
                    this.setPage(page);
                } else {
                    if (!this.dragging) {
                        this.currentPage = this.slideEls.length;
                    }
                    this._revert();
                }
            },
            setPage(page) {
                var propName, translateName;
                this.lastPage = this.currentPage;
                this.currentPage = page;
                this.page = page;


                if (this.isHorizontal()) {
                    propName = 'clientWidth';
                    translateName = 'translateX';
                } else {
                    propName = 'clientHeight';
                    translateName = 'translateY';
                }
                this[translateName] = -[].reduce.call(this.slideEls, function (total, el, i) {
                    return i - 1 > page - 2 ? total : total + el[propName];
                }, 0);

                this._onTransitionStart();
            },
            isHorizontal() {
                return this.direction === HORIZONTAL;
            },
            isVertical() {
                return this.direction === VERTICAL;
            },
            _onTouchStart(e) {
                this._fixPage();

                if (this.onlyExternal) {
                    return;
                }
                this.startPos = this._getTouchPos(e);
                this.delta = 0;
                this.startTranslateX = this.translateX;
                this.startTranslateY = this.translateY;
                this.startTime = new Date().getTime();
                this.dragging = true;
                this.stopAutoPlay();

                document.addEventListener('touchmove', this._onTouchMove, false);
                document.addEventListener('touchend', this._onTouchEnd, false);
                document.addEventListener('mousemove', this._onTouchMove, false);
                document.addEventListener('mouseup', this._onTouchEnd, false);
            },
            _onTouchMove(e) {
                this.delta = this._getTouchPos(e) - this.startPos;

                if (!this.loop && !this.allowSwipeToNext && this.delta < 0) {
                    return;
                }
                if (!this.loop && !this.allowSwipeToPrev && this.delta > 0) {
                    return;
                }


                if (!this.loop && (this.delta > 0 && this.isBeginning || this.delta < 0 && this.isEnd)) {
                    this.delta = this.delta * this.resistanceRatio;
                }


                if (!this.performanceMode) {
                    if (this.isHorizontal()) {
                        this.translateX = this.startTranslateX + this.delta;
                        this.$emit('slider-move', this.translateX);
                    } else {
                        this.translateY = this.startTranslateY + this.delta;
                        this.$emit('slider-move', this.translateY);
                    }
                }


                if (this.isVertical() || this.isHorizontal() && Math.abs(this.delta) > 0) {
                    e.preventDefault();
                }
            },
            _onTouchEnd(e) {


                var isQuickAction = new Date().getTime() - this.startTime < 1000;
                if (this.delta < -100 || (isQuickAction && this.delta < -15)) {
                    if (!this.allowSwipeToNext && this.dragging) {
                    } else {
                        this.next();
                    }
                } else if (this.delta > 100 || (isQuickAction && this.delta > 15)) {
                    if (!this.allowSwipeToPrev && this.dragging) {

                    } else {
                        this.prev();
                    }

                } else {
                    this._revert();
                }
                this.dragging = false;
                document.removeEventListener('touchmove', this._onTouchMove);
                document.removeEventListener('touchend', this._onTouchEnd);
                document.removeEventListener('mousemove', this._onTouchMove);
                document.removeEventListener('mouseup', this._onTouchEnd);
            },
            _onWheel(e) {
                if (this.mousewheelControl) {

                    // TODO Support apple magic mouse and trackpad.
                    if (!this.transitioning) {
                        if (e.deltaY > 0) {
                            this.next();
                        } else {
                            this.prev();
                        }
                    }

                    if (this._isPageChanged()) e.preventDefault();

                }
            },
            _revert() {
                this.setPage(this.currentPage);
            },
            _getTouchPos(e) {
                var key = this.isHorizontal() ? 'pageX' : 'pageY';
                return e.changedTouches ? e.changedTouches[0][key] : e[key];
            },
            _onTransitionStart() {

                this.transitioning = true;
                if (this._isPageChanged()) {
                    this.$emit('slide-change-start', this.currentPage);
                } else {
                    this.$emit('slide-revert-start', this.currentPage);
                }
            },
            _onTransitionEnd() {

                if (this.autoplayDisableOnInteraction == false && this.auto_play_stoped) {
                    this.startAutoPlay();
                } else if (!this.loop && this.autoplayStopOnLast && this.currentPage == this.slideEls.length) {
                    this.stopAutoPlay();
                }

                this.transitioning = false;
                if (this._isPageChanged()) {
                    this.$emit('slide-change-end', this.currentPage);
                } else {
                    this.$emit('slide-revert-end', this.currentPage);
                }
                this._fixPage();
            },
            _isPageChanged() {
                return this.lastPage !== this.currentPage;
            }
        }
    };
</script>
