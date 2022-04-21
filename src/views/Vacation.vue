<template>

  <div class="container-fluid" style="background: #FFFFFF;">
    <br>
    <div class="img-w">

    </div>


  </div>

</template>
<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
@Component

  export default class Welcome extends Vue {
    userName = "";
    // check = "";

    async mounted() {
      this.userName = this.$store.state.user.name;
      console.log(this.$store);
    }
  }
</script>
<style media="screen">
  .img-w {
    background-image: url("/assets/images/Image.png");
    height: 80vh;
    background-repeat: no-repeat;
    background-position: center top;
    padding-bottom: 10%;
  }

  .custom-h5{
    font-size: 2em !important;
    font-family: IBM Plex Sans Condensed;
    color: #50504f;
    font-weight: 600;
  }
</style>
