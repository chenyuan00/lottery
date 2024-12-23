<template>
  <div id="root">
    <header>
      <Publicity v-show="!running" />
      <el-button class="res" type="text" @click="showResult = true">
        抽奖结果
      </el-button>
      <el-dropdown class="con" @command="handleCommand">
        <span class="el-dropdown-link">
          配置菜单<i class="el-icon-arrow-down el-icon--right"></i>
        </span>
        <el-dropdown-menu slot="dropdown">
          <el-dropdown-item command="lotteryConfig">抽奖配置</el-dropdown-item>
          <el-dropdown-item command="reset">重置</el-dropdown-item>
          <el-dropdown-item command="importList">导入名单</el-dropdown-item>
          <el-dropdown-item command="importPhoto" >导入照片</el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
      <!-- <el-button class="con" type="text" @click="showConfig = true">
        抽奖配置
      </el-button> -->
    </header>
    <div id="main" :class="{ mask: showRes }"></div>
    <div id="tags">
      <ul v-for="item in datas" :key="item.key">
        <li>
          <a
            href="javascript:void(0);"
            :style="{
              color: '#fff',
            }"
          >
            {{ item.name ? item.name : item.key }}
            <img v-if="item.photo" :src="item.photo" :width="80" :height="80" />
          </a>
        </li>
      </ul>
    </div>
    <transition name="bounce">
      <div id="resbox" v-show="showRes">
        <p @click="showRes = false">{{ categoryName }}抽奖结果：</p>
        <div class="container">
          <span
            v-for="item in resArr"
            :key="item.key"
            class="itemres"
            :style="resCardStyle"
            :data-id="item.name"
            @click="showRes = false"
            :class="{
              numberOver:
                !!photos.find((d) => d.id === item.key) ||
                !!list.find((d) => d.key === item.key),
            }"
          >
            <span class="cont" v-if="!photos.find((d) => d.id === item.key)">
              <span
                v-if="!!list.find((d) => d.key === item.key)"
                :style="{
                  fontSize: '40px',
                }"
              >
                {{ list.find((d) => d.key === item.key).name }}
              </span>
              <span v-else>
                {{ item.key }}
              </span>
            </span>
            <img
              v-if="photos.find((d) => d.id === item.key)"
              :src="photos.find((d) => d.id === item.key).value"
              alt="photo"
              :width="200"
              :height="300"
            />
          </span>
        </div>
      </div>
    </transition>

    <el-button
      class="audio"
      type="text"
      @click="
        () => {
          playAudio(!audioPlaying);
        }
      "
    >
      <i
        
        :class="[audioPlaying ? 'el-icon-video-pause' : 'el-icon-video-play']"
      ></i>
    </el-button>

    <LotteryConfig :visible.sync="showConfig" @resetconfig="reloadTagCanvas" />
    <Tool 
      ref="tool"
      @toggle="toggle"
      @resetConfig="reloadTagCanvas"
      @getPhoto="getPhoto"
      :running="running"
      :closeRes="closeRes"
    />
    <Result :visible.sync="showResult"></Result>
    <audio
      ref="audio"
      id="audiobg"
      preload="auto"
      controls
      autoplay
      loop
      @play="playHandler"
      @pause="pauseHandler"
    >
      <source :src="audioSrc" />
      你的浏览器不支持audio标签
    </audio>
    <el-dialog
    :visible.sync="showRemoveoptions"
    width="300px"
    class="c-removeoptions"
    :append-to-body="true"
  >
    <el-form ref="form" :model="removeInfo" label-width="80px" size="mini">
      <el-form-item label="重置选项">
        <el-radio-group v-model="removeInfo.type">
          <el-radio border :label="0">重置全部数据</el-radio>
          <el-radio border :label="1">重置抽奖配置</el-radio>
          <el-radio border :label="2">重置名单</el-radio>
          <el-radio border :label="3">重置照片</el-radio>
          <el-radio border :label="4">重置抽奖结果</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="resetConfig">确定重置</el-button>
        <el-button @click="showRemoveoptions = false">取消</el-button>
      </el-form-item>
    </el-form>
  </el-dialog>
  <el-dialog
      :append-to-body="true"
      :visible.sync="showImport"
      class="import-dialog"
      width="400px"
    >
      <el-input
        type="textarea"
        :rows="10"
        placeholder="请输入对应的号码和名单(可直接从excel复制)，格式(号码 名字)，导入的名单将代替号码显示在抽奖中。如：
1 张三
2 李四
3 王五
				"
        v-model="listStr"
      ></el-input>
      <div class="footer">
        <el-button size="mini" type="primary" @click="transformList"
          >确定</el-button
        >
        <el-button size="mini" @click="showImport = false">取消</el-button>
      </div>
    </el-dialog>
    <Importphoto
    :visible.sync="showImportphoto"
    @getPhoto="getPhoto"
    @resetConfig="reloadTagCanvas"
  ></Importphoto>
  </div>
</template>
<script>
import {
  clearData,
  removeData,
} from '@/helper/index';
import LotteryConfig from '@/components/LotteryConfig';
import Publicity from '@/components/Publicity';
import Tool from '@/components/Tool';
import bgaudio from '@/assets/bg.mp3';
import beginaudio from '@/assets/begin.mp3';
import {
  getData,
  configField,
  resultField,
  newLotteryField,
  conversionCategoryName,
  listField,
} from '@/helper/index';
import { luckydrawHandler } from '@/helper/algorithm';
import Result from '@/components/Result';
import { database, DB_STORE_NAME } from '@/helper/db';
import Importphoto from '@/components/Importphoto';
export default {
  name: 'App',

  components: { LotteryConfig, Publicity, Tool, Result ,Importphoto},

  computed: {
    resCardStyle() {
      const style = { fontSize: '30px' };
      const { number } = this.config;
      if (number < 100) {
        style.fontSize = '100px';
      } else if (number < 1000) {
        style.fontSize = '80px';
      } else if (number < 10000) {
        style.fontSize = '60px';
      }
      return style;
    },
    config: {
      get() {
        return this.$store.state.config;
      },
    },
    result: {
      get() {
        return this.$store.state.result;
      },
      set(val) {
        this.$store.commit('setResult', val);
      },
    },
    list() {
      //导入的
      return this.$store.state.list;
    },
    allresult() {
      let allresult = [];
      for (const key in this.result) {
        if (this.result.hasOwnProperty(key)) {
          const element = this.result[key].map((item)=>{
            return item.key;
          });
          allresult = allresult.concat(element);
        }
      }
      return allresult;
    },
    datas() {
      const { number } = this.config;
      const nums = number >= 1500 ? 500 : this.config.number;
      const configNum = number > 1500 ? Math.floor(number / 3) : number;
      const randomShowNums = luckydrawHandler(configNum, [], nums);
      const randomShowDatas = randomShowNums.map((item) => {
        const listData = this.list.find((d) => d.key === item);
        const photo = this.photos.find((d) => d.id === item);
        return {
          key: item * (number > 1500 ? 3 : 1),
          name: listData ? listData.name : '',
          photo: photo ? photo.value : '',
        };
      });
      return randomShowDatas;
    },
    categoryName() {
      return conversionCategoryName(this.category);
    },
    photos() {
      return this.$store.state.photos;
    },
  },
  created() {
    const data = getData(configField);
    if (data) {
      this.$store.commit('setConfig', Object.assign({}, data));
    }
    const result = getData(resultField);
    if (result) {
      this.$store.commit('setResult', result);
    }

    const newLottery = getData(newLotteryField);
    if (newLottery) {
      const config = this.config;
      newLottery.forEach((item) => {
        this.$store.commit('setNewLottery', item);
        if (!config[item.key]) {
          this.$set(config, item.key, 0);
        }
      });
      this.$store.commit('setConfig', config);
    }

    const list = getData(listField);
    if (list) {
      this.$store.commit('setList', list);
    }
  },

  data() {
    return {
      listStr: '',
      running: false,
      showRes: false,
      showConfig: false,
      showResult: false,
      showRemoveoptions: false,
      showImport: false,
      showImportphoto: false,
      removeInfo: { type: 0 },
      resArr: [],
      category: '',
      audioPlaying: false,
      audioSrc: bgaudio,
    };
  },
  watch: {
    photos: {
      deep: true,
      handler() {
        this.$nextTick(() => {
          this.reloadTagCanvas();
        });
      },
    },
    showRemoveoptions(v) {
      if (!v) {
        this.removeInfo.type = 0;
      }
    },
    showImport(v){
      if(v==true) {
        if(this.list){
        let result =" ";
        this.list.forEach(item =>{
          result = result +item.key + " " + item.name + "\n";
         })
         this.listStr=result
      }
    }
  }
  },
  mounted() {
    this.startTagCanvas();
    setTimeout(() => {
      this.getPhoto();
    }, 1000);
    window.addEventListener('resize', this.reportWindowSize);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.reportWindowSize);
  },
  methods: {
    transformList() {
      const { listStr } = this;
      if (!listStr) {
        this.$message.error('没有数据');
        return
      }
      const list = [];
      const rows = listStr.split('\n');
      if (rows && rows.length > 0) {
        rows.forEach(item => {
          const rowList = item.split(/\t|\s/);
          if (rowList.length >= 2) {
            const key = Number(rowList[0].trim());
            const name = rowList[1].trim();
            key &&
              list.push({
                key,
                name
              });
          }
        });
      }
      if(list.length !=this.config.number) {
        this.$message({
        message: '导入人数与抽奖人数不匹配，请重新配置抽奖人数',
        type: 'warning',
      });
      this.showImport = false;
      }else{
        this.$store.commit('setList', list);

this.$message({
  message: '保存成功',
  type: 'success'
});
this.showImport = false;
this.$nextTick(() => {
  this.reloadTagCanvas();
});
      }
    
    },
    resetConfig() {
      const type = this.removeInfo.type;
      this.$confirm('此操作将重置所选数据，是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          switch (type) {
            case 0:
              clearData();
              this.$store.commit('setClearStore');
              database.clear(DB_STORE_NAME);
              break;
            case 1:
              removeData(configField);
              this.$store.commit('setClearConfig');
              break;
            case 2:
              removeData(listField);
              this.$store.commit('setClearList');
              break;
            case 3:
              database.clear(DB_STORE_NAME);
              this.$store.commit('setClearPhotos');
              break;
            case 4:
              removeData(resultField);
              this.$store.commit('setClearResult');
              break;
            default:
              break;
          }

          this.closeRes && this.closeRes();

          this.showRemoveoptions = false;
          this.$message({
            type: 'success',
            message: '重置成功!'
          });

          this.$nextTick(() => {
            this.$emit('resetConfig');
          });
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消'
          });
        });
    },
    handleCommand(command) {
        if(command=='lotteryConfig'){
          this.showConfig = true
        }else if(command=='reset'){
          this.showResult=false
          this.showConfig = false
          this.showImport = false
          this.showImportphoto = false
          
          this.showRemoveoptions=true
        }else if(command=='importList'){
          this.showImport = true
        }else if(command=='importPhoto'){
          this.showImportphoto = true
        }
    },
    reportWindowSize() {
      const AppCanvas = this.$el.querySelector('#rootcanvas');
      if (AppCanvas.parentElement) {
        AppCanvas.parentElement.removeChild(AppCanvas);
      }
      this.startTagCanvas();
    },
    playHandler() {
      this.audioPlaying = true;
    },
    pauseHandler() {
      this.audioPlaying = false;
    },
    playAudio(type) {
      if (type) {
        this.$refs.audio.play();
      } else {
        this.$refs.audio.pause();
      }
    },
    loadAudio() {
      this.$el.querySelector('#audiobg').load();
      this.$nextTick(() => {
        this.$el.querySelector('#audiobg').play();
      });
    },
    getPhoto() {
      database.getAll(DB_STORE_NAME).then((res) => {
        if (res && res.length > 0) {
          this.$store.commit('setPhotos', res);
        }
      });
    },
    speed() {
      return [0.1 * Math.random() + 0.01, -(0.1 * Math.random() + 0.01)];
    },
    createCanvas() {
      const canvas = document.createElement('canvas');
      canvas.width = document.body.offsetWidth;
      canvas.height = document.body.offsetHeight;
      canvas.id = 'rootcanvas';
      this.$el.querySelector('#main').appendChild(canvas);
    },
    startTagCanvas() {
      this.createCanvas();
      const { speed } = this;
      window.TagCanvas.Start('rootcanvas', 'tags', {
        textColour: null,
        initial: speed(),
        dragControl: 1,
        textHeight: 20,
        noSelect: true,
        lock: 'xy',
      });
    },
    reloadTagCanvas() {
      window.TagCanvas.Reload('rootcanvas');
    },
    closeRes() {
      this.showRes = false;
    },
    toggle(form) {
      const { speed, config } = this;
      if (this.running) {
        this.audioSrc = bgaudio;
        this.loadAudio();

        window.TagCanvas.SetSpeed('rootcanvas', speed());
        this.showRes = true;
        this.running = !this.running;
        this.$nextTick(() => {
          this.reloadTagCanvas();
        }); 

        
      } else {
        if (form) {
          setTimeout(() => {
           this.$refs.tool.startHandler()
             }, form.time*1000);
        }
        this.showRes = false;
        if (!form) {
          return;
        }

        this.audioSrc = beginaudio;
        this.loadAudio();

        const { number } = config;
        const { category, mode, qty, remain, allin } = form;
        let num = 1;
        if (mode === 1 || mode === 5) {
          num = mode;
        } else if (mode === 0) {
          num = remain;
        } else if (mode === 99) {
          num = qty;
        }
        const resArr = luckydrawHandler(
          number,
          allin ? [] : this.allresult,
          num
        );
        const newResArr= resArr.map((item)=>{
          const findName=this.list.find(o => o.key==item)
          if(findName){
            return{
              key:findName.key,
              name:findName.name
            }
          }
        })
        this.resArr = newResArr;
        this.category = category;

        if (!this.result[category]) {
          this.$set(this.result, category, []);
        }
        const _result={}
        for(const key in this.result){
          if(this.result.hasOwnProperty(key)){
            _result[key] = this.result[key].map((item)=>{
                return item.key
            })
          }
        }
        const oldRes = _result[category] || [];

        const data = Object.assign({}, _result, {
          [category]: oldRes.concat(resArr)
        });
        const newData={}
        for(const key in data){
          if(data.hasOwnProperty(key)){
            newData[key] = data[key].map((item)=>{
                const findName=this.list.find(o => o.key==item)
                  if(findName){
                    return {
                      key:findName.key,
                      name:findName.name
                    }
                  }
            })
          }
        }

        this.result = newData;
        window.TagCanvas.SetSpeed('rootcanvas', [3, 0.5]);
        this.running = !this.running;
      }
    },
  }
};
</script>
<style lang="scss">
i.el-icon-video-pause {
  font-size: 43px;
}
i.el-icon-video-play {
  font-size: 43px;
}
#root {
  height: 100%;
  position: relative;
  background-image: url('./assets/bg3.png');
  background-size: 100% 100%;
  background-position: center center;
  background-repeat: no-repeat;
  background-color: #121936;
  .mask {
    -webkit-filter: blur(5px);
    filter: blur(5px);
  }
  header {
    height: 50px;
    line-height: 50px;
    position: relative;
    .el-button {
      position: absolute;
      top: 17px;
      padding: 0;
      z-index: 9999;
      &.con {
        right: 20px;
      }
      &.res {
        right: 100px;
      }
    }
    .el-dropdown {
      color: #409eff;
      position: absolute;
      top: -2px;
      padding: 0;
      z-index: 9999;
      &.con {
        right: 12px;
      }
    }
  }
  .audio {
    position: absolute;
    top: 100px;
    right: 30px;
    width: 40px;
    height: 40px;
    line-height: 40px;
    padding: 0;
    text-align: center;
    .iconfont {
      position: relative;
      left: 1px;
    }
    
  }
  .copy-right {
    position: absolute;
    right: 0;
    bottom: 0;
    color: #ccc;
    font-size: 12px;
  }
  .bounce-enter-active {
    animation: bounce-in 1.5s;
  }
  .bounce-leave-active {
    animation: bounce-in 0s reverse;
  }
}
#main {
  height: 100%;
}

#resbox {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1280px;
  transform: translateX(-50%) translateY(-50%);
  text-align: center;
  p {
    color: #ffd700;
    font-size: 50px;
    line-height: 120px;
  }
  .container {
    display: flex;
    justify-content: center;
    flex-wrap: wrap;
  }
  .itemres {
    background: #fff;
    width: 200px;
    height: 300px;
    border-radius: 4px;
    border: 1px solid #ccc;
    line-height: 160px;
    font-weight: bold;
    margin-right: 20px;
    margin-bottom: 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    .cont {
      display: flex;
      justify-content: center;
      align-items: center;
    }
    &.numberOver::before {
      content: attr(data-id);
      width: 70px;
      height: 22px;
      line-height: 22px;
      background-color: #fff;
      position: absolute;
      bottom: 0;
      left: 0;
      font-size: 14px;
      // border-radius: 50%;
      z-index: 1;
    }
  }
}
</style>
