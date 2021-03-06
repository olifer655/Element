<script>
  // 包装 fetch API
  const $fetch = (url) => fetch(url)
    .then(res => {
      const json = res.json();
      if (res.status >= 200 && res.status < 300) return json;
      return json.then(Promise.reject.bind(Promise));
    });

  export default {
    props: {
      timeout: {
        type: Number,
        default: 5000
      },

      apihost: {
        type: String,
        default: '//m.ele.me/restapi'
      },

      show: {
        type: Boolean,
        default: true
      },

      tips: {
        type: Object,
        default: () => ({
          loading: '正在定位，请稍后...',
          failed: '定位失败，点此重新定位',
          fallback: '您附近的餐厅'
        })
      },

      resolve: {
        type: Function,
        default() {}
      },

      reject: {
        type: Function,
        default() {}
      }
    },

    data: () => ({
      geohash: '',
      locName: '',

      navHash: '',
      apiHash: '',
      apiLock: false,
      isTrying: null // interval ID
    }),

    computed: {
      locTips() {
        return this.isTrying ? this.tips.loading : this.tips.failed;
      }
    },

    watch: {
      // 一旦获取到 geohash，清空计时器，执行 resolve，获取地址名
      geohash(val) {
        if (!val) return;
        clearTimeout(this.isTrying);

        this.resolve(val);
        this.getLocName();
      }
    },

    methods: {
      // 获取地址名称
      getLocName() {
        if (!this.show) return;

        $fetch(this.apihost + '/v2/pois/' + this.geohash)
          .then(json => {
            this.locName = json.name;
          })
          .catch(() => {
            this.locName = this.tips.fallback;
          });
      },

      // 获取 restAPI 地址
      getApiHash() {
        if (this.apiLock) return;
        this.apiLock = true;

        $fetch(this.apihost + '/v1/cities?type=guess')
          .then(json => {
            this.apiHash = Geohash.encode(json.latitude, json.longitude);
          })
          .catch(() => {
            this.apiLock = false;
          });
      },

      // 获取浏览器地址
      getNavigatorHash() {
        if (!navigator.geolocation || this.navHash) return;

        navigator.geolocation.getCurrentPosition(position => {
          this.navHash = Geohash.encode(position.coords.latitude, position.coords.longitude);
        });
      },

      // 获取 App 地址
      getHybridHash() {
        hybridAPI.getGlobalGeohash(geohash => {
          this.geohash = geohash;
        });
      },

      // 选择最佳结果：Hybrid API > Navigator > restAPI
      selectBestLoc() {
        clearTimeout(this.isTrying);
        this.isTrying = null;

        if (this.geohash) return;
        if (this.navHash) return this.geohash = this.navHash;
        if (this.apiHash) return this.geohash = this.apiHash;

        this.reject();
      },

      // 每隔 500 毫秒请求一次 Hybrid API 数据
      // Hybird API 有坑，回调函数必然执行，但未取得时参数为空
      loopTryHybrid() {
        if (this.geohash || this.isTrying) return;

        this.isTrying = setInterval(this.getHybridHash, 500);
        setTimeout(this.selectBestLoc, this.timeout);
      },

      // App 内部取地址模式
      useAppMode() {
        const fallback = () => {
          if (this.geohash) return;
          this.getNavigatorHash();
          this.getApiHash();
        };

        this.getHybridHash();
        this.loopTryHybrid();

        setTimeout(fallback, this.timeout / 2);
      },

      // 外部取地址模式
      useExternalMode() {
        this.isTrying = setTimeout(this.selectBestLoc, this.timeout);

        this.getNavigatorHash();
        this.getApiHash();
        this.$watch('navHash', val => {
          this.geohash = val;
        });
      },

      // 选择模式
      getUserLoc() {
        if (this.geohash) return;
        if (/Eleme/i.test(navigator.userAgent)) return this.useAppMode();
        this.useExternalMode();
      }
    },

    events: {
      initCompass(geohash) {
        if (geohash) return this.geohash = geohash;
        this.getUserLoc();
      }
    }
  };
</script>

<template>
  <div class="compass-wrap" v-show="show" @click="getUserLoc">
    <svg class="compass-location" v-if="geohash"
      viewBox="0 0 64 64" xmlns="http://www.w3.org/2000/svg"><path d="M32.15.304C18.47.304 7.337 11.478 7.337 25.212c0 18.856 22.126 37.26 23.067 38.034a2.743 2.743 0 0 0 3.492 0c.943-.773 23.067-19.178 23.067-38.034C56.963 11.478 45.833.304 32.15.304zm0 31.066c-5.086 0-9.21-4.112-9.21-9.185S27.064 13 32.15 13c5.088 0 9.212 4.112 9.212 9.185s-4.124 9.185-9.21 9.185h-.002z" fill-rule="evenodd"/></svg>
    <svg class="compass-refresh" :class="[ isTrying ? 'compass-loading': '' ]" v-else
      viewBox="0 0 64 64" xmlns="http://www.w3.org/2000/svg"><path d="M57.927 33.253c.02-.43.032-.86.032-1.293C57.96 17.623 46.335 6 32 6a25.82 25.82 0 0 0-13.617 3.86 2.995 2.995 0 1 0 3.137 5.092A19.875 19.875 0 0 1 32 11.982c11.033 0 19.976 8.945 19.976 19.978 0 .434-.015.865-.042 1.293h-5.808l8.937 12.798L64 33.254h-6.073zM44.2 48.475c-.64 0-1.232.2-1.72.543A19.875 19.875 0 0 1 32 51.988c-11.033 0-19.976-8.945-19.976-19.978 0-.434.015-.865.042-1.293h5.808L8.937 17.92 0 30.716h6.073c-.02.43-.032.86-.032 1.293 0 14.337 11.624 25.96 25.96 25.96a25.82 25.82 0 0 0 13.617-3.86 2.995 2.995 0 0 0-1.417-5.635z" fill-rule="evenodd"/></svg>

    <span class="compass-name" v-if="geohash">{{ locName }}</span>
    <span class="compass-tips" v-else>{{ locTips }}</span>
  </div>
</template>

<style src="./compass.css"></style>
