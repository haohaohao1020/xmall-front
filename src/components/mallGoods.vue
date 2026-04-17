<template>
  <div class="good-item">
    <div style="">
      <div class="good-img">
        <a @click="openProduct(msg.productId)">
          <img v-lazy="msg.productImageBig" :alt="msg.productName" :key="msg.productImageBig">
        </a>
        <!-- 收藏按钮 -->
        <div class="favorite-btn" @click.stop="toggleFavorite">
          <i class="el-icon-star-on" v-if="isFavorite"></i>
          <i class="el-icon-star-off" v-else></i>
        </div>
      </div>
      <h6 class="good-title" v-html="msg.productName">{{msg.productName}}</h6>
      <h3 class="sub-title ellipsis">{{msg.subTitle}}</h3>
      <div class="good-price pr">
        <div class="ds pa">
          <a @click="openProduct(msg.productId)">
            <y-button text="查看详情" style="margin: 0 5px"></y-button>
          </a>
          <y-button text="加入购物车"
                    style="margin: 0 5px"
                    @btnClick="addCart(msg.productId,msg.salePrice,msg.productName,msg.productImageBig)"
                    classStyle="main-btn"
          ></y-button>
        </div>
        <p><span style="font-size:14px">￥</span>{{Number(msg.salePrice).toFixed(2)}}</p>
      </div>
    </div>
  </div>
</template>
<script>
  import YButton from '/components/YButton'
  import { addCart, addFavorite, delFavorite, checkFavorite } from '/api/goods.js'
  import { mapMutations, mapState } from 'vuex'
  import { getStore, setStore } from '/utils/storage'
  export default {
    props: {
      msg: {
        salePrice: 0
      }
    },
    data () {
      return {
        isFavorite: false
      }
    },
    created () {
      this.checkFavorite()
    },
    watch: {
      'msg.productId' (newVal) {
        if (newVal) {
          this.checkFavorite()
        }
      }
    },
    methods: {
      ...mapMutations(['ADD_CART', 'ADD_ANIMATION', 'SHOW_CART']),
      goodsDetails (id) {
        this.$router.push({path: 'goodsDetails/productId=' + id})
      },
      openProduct (id) {
        window.open('//' + window.location.host + '/#/goodsDetails?productId=' + id)
      },
      addCart (id, price, name, img) {
        if (!this.showMoveImg) {     // 动画是否在运动
          if (this.login) { // 登录了 直接存在用户名下
            addCart({userId: getStore('userId'), productId: id, productNum: 1}).then(res => {
              // 并不重新请求数据
              this.ADD_CART({productId: id, salePrice: price, productName: name, productImg: img})
            })
          } else { // 未登录 vuex
            this.ADD_CART({productId: id, salePrice: price, productName: name, productImg: img})
          }
          // 加入购物车动画
          var dom = event.target
          // 获取点击的坐标
          let elLeft = dom.getBoundingClientRect().left + (dom.offsetWidth / 2)
          let elTop = dom.getBoundingClientRect().top + (dom.offsetHeight / 2)
          // 需要触发
          this.ADD_ANIMATION({moveShow: true, elLeft: elLeft, elTop: elTop, img: img})
          if (!this.showCart) {
            this.SHOW_CART({showCart: true})
          }
        }
      },
      // 检查是否已收藏
      checkFavorite () {
        // 先检查本地缓存
        let favorites = getStore('favorites') || '[]'
        favorites = JSON.parse(favorites)
        this.isFavorite = favorites.some(item => item.productId === this.msg.productId)

        // 如果已登录，从服务器获取收藏状态
        if (this.login) {
          let params = {
            params: {
              userId: getStore('userId'),
              productId: this.msg.productId
            }
          }
          checkFavorite(params).then(res => {
            if (res.success) {
              this.isFavorite = res.result.isFavorite
              // 更新本地缓存
              if (this.isFavorite) {
                if (!favorites.some(item => item.productId === this.msg.productId)) {
                  favorites.push({
                    productId: this.msg.productId,
                    productName: this.msg.productName,
                    productImageBig: this.msg.productImageBig,
                    salePrice: this.msg.salePrice,
                    subTitle: this.msg.subTitle,
                    addTime: new Date().getTime()
                  })
                  setStore('favorites', JSON.stringify(favorites))
                }
              }
            }
          }).catch(() => {
            // 使用本地数据
          })
        }
      },
      // 切换收藏状态
      toggleFavorite () {
        if (!this.login) {
          this.$message.warning('请先登录后再收藏商品')
          this.$router.push('/login')
          return
        }

        let favorites = getStore('favorites') || '[]'
        favorites = JSON.parse(favorites)
        const index = favorites.findIndex(item => item.productId === this.msg.productId)
        
        if (index > -1) {
          // 取消收藏 - 调用接口
          let params = {
            userId: getStore('userId'),
            productId: this.msg.productId
          }
          delFavorite(params).then(res => {
            if (res.success) {
              favorites.splice(index, 1)
              this.isFavorite = false
              setStore('favorites', JSON.stringify(favorites))
              this.$message.success('已取消收藏')
            } else {
              this.$message.error(res.message || '取消收藏失败')
            }
          }).catch(() => {
            // 接口失败，本地处理
            favorites.splice(index, 1)
            this.isFavorite = false
            setStore('favorites', JSON.stringify(favorites))
            this.$message.success('已取消收藏')
          })
        } else {
          // 添加收藏 - 调用接口
          let params = {
            userId: getStore('userId'),
            productId: this.msg.productId,
            productName: this.msg.productName,
            productImageBig: this.msg.productImageBig,
            salePrice: this.msg.salePrice
          }
          addFavorite(params).then(res => {
            if (res.success) {
              favorites.push({
                productId: this.msg.productId,
                productName: this.msg.productName,
                productImageBig: this.msg.productImageBig,
                salePrice: this.msg.salePrice,
                subTitle: this.msg.subTitle,
                addTime: new Date().getTime()
              })
              this.isFavorite = true
              setStore('favorites', JSON.stringify(favorites))
              this.$message.success('收藏成功')
            } else {
              this.$message.error(res.message || '收藏失败')
            }
          }).catch(() => {
            // 接口失败，本地处理
            favorites.push({
              productId: this.msg.productId,
              productName: this.msg.productName,
              productImageBig: this.msg.productImageBig,
              salePrice: this.msg.salePrice,
              subTitle: this.msg.subTitle,
              addTime: new Date().getTime()
            })
            this.isFavorite = true
            setStore('favorites', JSON.stringify(favorites))
            this.$message.success('收藏成功')
          })
        }
      }
    },
    computed: {
      ...mapState(['login', 'showMoveImg', 'showCart'])
    },
    mounted () {
    },
    components: {
      YButton
    }
  }
</script>
<style lang="scss" rel="stylesheet/scss" scoped>
  @import "../assets/style/mixin";
  @import "../assets/style/theme";

  .good-item {
    background: #fff;
    width: 25%;
    transition: all .5s;
    height: 430px;
    &:hover {
      transform: translateY(-3px);
      box-shadow: 1px 1px 20px #999;
      .good-price p {
        display: none;
      }
      .ds {
        display: flex;
      }
    }
    .ds {
      @extend %block-center;
      width: 100%;
      display: none;
    }

    .good-img {
      position: relative;
      img {
        margin: 50px auto 10px;
        @include wh(206px);
        display: block;
      }
      .favorite-btn {
        position: absolute;
        top: 10px;
        right: 10px;
        width: 36px;
        height: 36px;
        background: rgba(255, 255, 255, 0.9);
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        cursor: pointer;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        transition: all 0.3s;
        z-index: 10;
        &:hover {
          transform: scale(1.1);
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
        }
        i {
          font-size: 18px;
          color: #999;
          transition: all 0.3s;
        }
        i.el-icon-star-on {
          color: #ff6700;
        }
        &:hover i {
          color: #ff6700;
        }
      }
    }
    .good-price {
      margin: 15px 0;
      height: 30px;
      text-align: center;
      line-height: 30px;
      color: #d44d44;
      font-family: Arial;
      font-size: 18px;
      font-weight: 700;
    }
    .good-title {
      line-height: 1.2;
      font-size: 16px;
      color: #424242;
      margin: 0 auto;
      padding: 0 14px;
      text-align: center;
      overflow: hidden;
    }
    h3 {
      text-align: center;
      line-height: 1.2;
      font-size: 12px;
      color: #d0d0d0;
      padding: 10px;
    }

  }
</style>
