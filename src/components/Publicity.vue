<template>
  <div class="c-Publicity">
    <div style="height: 50px;">
      <div class="item" :class="actiname">
        <span class="title"> {{ title }}</span>
      </div>
    </div>
    <!-- <el-carousel
      height="50px"
      :autoplay="true"
      indicator-position="none"
      arrow="never"
      :interval="3000"
    >
      <el-carousel-item v-for="item in message" :key="item.key">
        <div class="item" :class="{ actiname: item.key === 0 }">
          <span v-if="item.title" class="title"> {{ item.title }}</span>
          <span v-if="item.value" class="value">
            {{ item.value }}
          </span>
        </div>
      </el-carousel-item>
    </el-carousel> -->
  </div>
</template>
<script>
import { conversionCategoryName } from '@/helper/index';

export default {
  name: 'Publicity',
  computed: {
    config() {
      return this.$store.state.config;
    },
    title() {
      return this.$store.state.config.name;
    },
    result() {
      return this.$store.state.result;
    },
    message() {
      const { result, config } = this;
      const fields = Object.keys(config);

      let message = [{ key: 0, title: config.name }];
      fields.forEach((item, index) => {
        let label = conversionCategoryName(item);
        if (result[item] && config[item] > 0) {
          message.push({
            key: index + 1,
            title: `${label}抽奖结果:`,
            value: `${
              result[item].length > 0 ? result[item].join('、') : '暂未抽取'
            }`
          });
        }
      });

      return message;
    }
  }
};
</script>
<style lang="scss">
.c-Publicity {
  height: 100%;
  // width: 1000px;
  background-color: rgba(255, 255, 255, 0.1);
  margin: 0 auto;
  position: relative;
  overflow: hidden;
  .el-carousel {
    width: 80vw;
    margin: 0 auto;
  }
  .item {
    font-weight: bold;
    text-align: center;
    color: black;
    font-size: 28px;
    .title {
      color: #ffff01;
    }
    .value {
      margin-left: 10px;
    }
    &.actiname {
      .title {
        color: red;
        font-size: 20px;
      }
    }
  }
}
</style>
