<template>
  <div class="modal">
    <div class="container">
      <div class="alert">
        <img src="../assets/images/alert.svg" width="135px">
      </div>

      <p class="message"
      v-for="(message, index) in messages" :key="index">
        {{ message }}
      </p>

      <div class="btn-container">
        <div>
          <button class="btn"
          @click="act">{{ btnMsg }}</button>
        </div>
        <div>
          <button class="btn btn-sub"
          @click="hide">Back to current game</button>
        </div>   
      </div>
    </div>
  </div>
</template>

<script lang='ts'>
import { Component, Vue, Prop, Emit } from 'vue-property-decorator';

@Component({})

export default class Modal extends Vue {
  @Prop(Array) private messages;
  @Prop(String) private btnMsg;

  @Emit() public act () {
    return;
  }
  @Emit() public hide () {
    return;
  }
}
</script>

<style lang='scss' scoped >
@import '../assets/all';

.modal {
  width: 100%;
  height: 100%;
  font-weight: 900;
  background-color: rgba(1, 1, 1, .7);
  display: inline-flex;
  justify-content: center;
  align-items: center;
}
.container {
  padding: 50px 115px;
  background-color: #ffffff;
  border-radius: 20px;
  text-align: center;
}
.alert {
  margin-bottom: 10px;
}
.message {
  line-height: 1.5;
}
.btn-container {
  margin-top: 30px;
}
</style>
