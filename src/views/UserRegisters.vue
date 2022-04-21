<template>
  <div>
    <div class="row">
      <div class="col-md-7">
        <h6 class="text-muted">Registros por colaborador</h6>
        <div class="row">
         <div class="col-md-6">
           <a-select  default-value="Selecciona un colaborador" style="width: 100%" @change="userChange"  placeholder="Selecciona un proyecto">
             <a-select-option  value="">
               Todos los colaboradores
             </a-select-option>
             <a-select-option  v-for="(user, index) in users" :key='index'>
               {{ user.name }}
             </a-select-option>
             template
           </a-select>
         </div>
        </div>
      </div>
      <div class="col-md-5 text-right" style="border-bottom: 1px solid #F5B133">
        <a-range-picker  @change="dateRangeSelected" />
      </div>
    </div>
    <div class="row">
      <rotate-square2 v-if="isLoading"></rotate-square2>
    </div>

    <highcharts v-show="data.length" id="myChart"  :options="chartOptions" ref="chart"></highcharts>
    <div class="row mt-2 p-3" v-if="data.length">
      <div class="row p-3">
        <button class="btn btn-success" @click="exportToExcel">Exportar</button>

       <!-- <button class="btn btn-danger ml-2" v-if="!isLoadingFormat" @click="generatePdf">Generar PDF</button>-->

<!--        <rotate-square2 v-if="isLoadingFormat"></rotate-square2>-->

<!--        <a :href="path" v-if="successReport" class="btn btn-warning ml-2" target="_blageneratePdfnk">Descargar Aqui</a>
        <a v-if="errorReport" class="text-danger">No se pudo generar el reporte</a>-->
      </div>
      <table class="table table-bordered table-responsive">
        <tr>
          <th>#</th>
          <th>Colaborador</th>
          <th>Posición</th>
          <th>Código de la actividad</th>
          <th>Descripción</th>
          <th>Inicio</th>
          <th>Fin</th>
          <th class="text-center">Tiempo (horas)</th>
        </tr>
        <tr v-for="(item, index) in data" :key='index' :class="{ overlapping : item.overlapping }">
          <td>{{ index + 1}}</td>
          <td>{{ item.colaborador }}</td>
          <td>{{ item.cargo }}</td>
          <td>{{ item.codigo_proyecto }}</td>
          <td>{{ item.descripcion }}</td>
          <td>{{ item.dateStart}}</td>
          <td>{{ item.dateEnd}}</td>
          <td>{{ item.tiempo_estimado }}</td>
        </tr>
        <tr>
          <td colspan="8" class="text-right pr-4"><h5>Total: {{ total }} Hrs</h5></td>
        </tr>
      </table>
    </div>
  </div>
</template>

<script lang="ts">
import {Component, Vue} from "vue-property-decorator";
import {RotateSquare2} from "vue-loading-spinner";
import moment from "moment";
import axios from "axios";
import XLSX from "xlsx";

@Component({
  components:{
    RotateSquare2
  }
})

export default class UsersRegisters extends Vue {
  private isLoading = false;
  private isLoadingFormat = false;
  private series: Array<any> = [];
  private categories: Array<any> = [];
  private data: any = [];
  private users: any = [];
  private user: any = null;
  private startDate: any = null;
  private endDate: any = null;
  private total = 0;
  private year: any = null;
  private month: any = null;

  successReport = false;
  errorReport = false;
  path = "";

  async created(){
    const response = await axios.post('/api/user-list');

    this.users = response.data;
  }

  async generatePdf(){
    console.log(this.user);
    const params = {
      "startDate" : this.startDate,
      "endDate" : this.endDate,
      "userId" : this.user.id
    };
    this.isLoadingFormat = true;
    try{
      const response = await axios.post('/api/user-registers-report', params);
      if (response.data.success){
        this.path =  response.data.path;
        this.successReport = true;
        this.errorReport = false;
      }else{
        this.path = "";
        this.successReport = false;
        this.errorReport = true;
      }
      this.isLoadingFormat = false;
    }catch (e) {
      this.errorReport = true;
      this.isLoadingFormat = false;
    }

  }

  private userChange(index: any){
    this.user =  (typeof index === 'number') ? this.users[index] : null;
    this.getRegistersByDateRange(this.startDate, this.endDate);
  }

  private dateRangeSelected(date: any){
    if (!date.length) return;
    this.startDate = moment(date[0]._d).format('YYYY-MM-DD');
    this.endDate = moment(date[1]._d).format('YYYY-MM-DD');
    const month = moment(date[0]._d).month() + 1;
    this.month = (month).toLocaleString('en-US', {minimumIntegerDigits: 2, useGrouping:false})
    this.year = moment(date[1]._d).year();
    this.getRegistersByDateRange(this.startDate, this.endDate);
  }

  async getRegistersByDateRange(start: string, end: string){
    if (this.user == null) return;
    if (start == null && end ==null) return;
    this.path = "";
    this.successReport = false;
    this.errorReport = false;
    try{
      this.data = [];
      this.isLoading = true;
      const result = await axios.post('/api/user-registers-list', { start : start, end : end, "user_id" : this.user.id });
      this.total = result.data.total;

      this.checkIfThereAreRegisterInTheSameTime( result.data.data);

      const resultCharts = await axios.post('/api/user-registers-chart', { start : start, end : end, "user_id" : this.user.id });
      this.categories = resultCharts.data.categories;
      this.series = resultCharts.data.series;
      console.log(resultCharts);

      this.isLoading = false;
    }catch (e) {
      console.log(e);
      this.isLoading = false;
    }
  }

  checkIfThereAreRegisterInTheSameTime(data: any){
    const result = data.map((item: any) => {
      const startDate = moment(item.dateStart);
      const endDate =  moment(item.dateEnd);

      const find = data.find( (value: any) => {
        const start= moment(value.dateStart);
        const end =  moment(value.dateEnd);
        return (startDate.isBetween(start, end) || endDate.isBetween(start, end));
      })
      item.overlapping = !!(find);
      return item;
    })

    this.data = result;
  }

  get chartOptions() {
    return {
      chart: {
        type: 'bar'
      },
      title: {
        text: 'Distribución del tiempo reportado, por actividad'
      },
      xAxis: {
        categories: this.categories
      },
      yAxis: {
        min: 0,
        title: {
          text: ''
        }
      },
      legend: {
        reversed: true
      },
      plotOptions: {
        series: {
          stacking: 'normal'
        }
      },
      series: [
        this.series
      ]
    };
  }

  exportToExcel(){
    const headers = ['colaborador', 'cargo', 'codigo_proyecto', 'descripcion', 'dateStart', 'dateEnd', 'tiempo_estimado' ];
    const data = XLSX.utils.json_to_sheet( this.data, {
      header: headers
    })
    data['A1'].v = 'Colaborador'
    data['B1'].v = 'Cargo'
    data['C1'].v = 'Código del Proyecto'
    data['D1'].v = 'Descripción'
    data['E1'].v = 'Inicio'
    data['F1'].v = 'Fin'
    data['G1'].v = 'Tiempo Estimado (hrs)'

    const userName = this.user.name.replace(/ /g, "_");
    const workbook = XLSX.utils.book_new()
    const workbookNname = "registros";
    XLSX.utils.book_append_sheet(workbook, data, workbookNname);
    const fileName = `${this.year}_${this.month}_${userName}.xlsx`;
    XLSX.writeFile(workbook, fileName);
  }
}
</script>

<style scoped>
#myChart{
  height: 800px !important;
}
.overlapping{
  background-color: #b53d00;
  color: white;
}
</style>
