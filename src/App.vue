<script setup>
import BaseListbox from "./components/BaseListbox.vue";
import {ref, watch, nextTick} from "vue";
import grafica from './components/grafica.vue'

///INICIALIZACION GRAFICAS CANVASJS
    const datosCargados=ref(false)
    const puntosAGraficar=ref(new Array());
    var puntos= new Array();
    
    var lapsoDias=14; //numero de días que queremos representar
    var stepTime=30; //minutos entre medidas
    
    var lapso=lapsoDias*24*60/stepTime //lapso en numero de registros a leer del fichero
    
    var lineas = new Array();
    
//////FIN INICIALIZACION GRAFICAS


const baseUrlGruporec="http://gruporec.csic.es/"
const PROYECTOS = Object.freeze({
  DIGIORCHARD: {value: 1, label: "DigiOrchard",url:'./Digiorchard/Datos'},
  ISADORA: {value: 2, label: "Isadora",url:'./Isadora'},
  IRRIWELL: {value: 3, label: "Irriwell",url:'./Irriwell/Datos'},
  WATCHPLANT: {value: 4, label: "Watchplant",url:'./Digiorchard/Datos'},
})
const proyectosREC = Object.values(PROYECTOS).map(({ value, label, url }) => ({ value, label, url }));

//usaremos este array para popular el 2do listbox
const archivosProyecto = ref([])
//usaremos este array para popular el 3er listbox
const vblesArchivo= ref([])

const archivos =[]

const form = ref({
  project_id: null,
  file_id: null,
  vble_id: null
});
const seleccionadoArchivo=ref(false)
const seleccionadaVble=ref(false)
const listadoFicheros = ref([])

const nuevoValor=()=>{console.log("seleccion realizada")}
watch(seleccionadoArchivo, nuevoValor)

//Cuando se selecciona un proyecto se llama a esta función y en form.value.project_id
//se tiene disponible el value del proyecto elegido
const recibidoProyecto=(sel)=>{
  seleccionadoArchivo.value = sel
    //console.log("sel: ", sel)
    //console.log("seleccion recibida. Seleccionado: ",form.value.project_id)
    //extraemos la url del proyecto:
    //console.log("proyectosREC: ",proyectosREC)
    let relativeURL=proyectosREC.find(proyecto => proyecto.value === form.value.project_id).url
    let url=`https://gruporec2.csic.es/nginx/listar.php?path=`+relativeURL
    //console.log("nueva url: ",url)
    fetchListFiles(url)
    }
  const recibidoArchivo=(sel)=>{
    seleccionadaVble.value = sel
    //let miFichero=listadoFicheros.find(file_id => file_id.value === form.value.file_id)
    // console.log(archivosProyecto.value)
    // console.log(form.value.file_id)
    // console.log(archivosProyecto.value[form.value.file_id])
    const filesProyecto = Object.values(archivosProyecto.value).map(({ value, label }) => ({ value, label }));

    let fileName=filesProyecto.find(file => file.value === form.value.file_id).label
    let relativePath=proyectosREC.find(proyecto => proyecto.value === form.value.project_id).url
    let fullPath=baseUrlGruporec+relativePath.replace("./","")+"/"
    let file=fileName+".dat"
    let url=`https://gruporec2.csic.es/nginx/leeDatosGruporec.php?path=`+fullPath+"&file="+file
    console.log("url: ",url)
    descargaDatos(url)
  }
  const recibidaVble= async (sel)=>{
    console.log("sel: ", sel)
    console.log("seleccion recibida. Seleccionado: ",form.value.vble_id)
    const vblesObj = Object.values(vblesArchivo.value).map(({ value, label }) => ({ value, label }));
    let vble= vblesObj.find(vble => vble.value === form.value.vble_id).label
    console.log("vble seleccionada: ", vble)
    puntosAGraficar.value=puntos[form.value.vble_id-1]
    console.log("nuevos puntos a graficar: ", puntosAGraficar.value)
    datosCargados.value=false;
    await nextTick()
    datosCargados.value = true;
    //datosCargados.value=true  
  }

async function descargaDatos(url){
  const res = await fetch(url)
  const datos=await res.text()
  //console.log("datos:",datos)
  trataDatos(datos)
  //const datos=await res.json()
  //console.log("datos:",datos)
}

function trataDatos(datos)
    {
    //console.log(datos)
    lineas=datos.split(/\r\n|\r|\n/g);
    const cabecera=lineas[1].replace(/"/g,'').split(',')
    popularVbles(cabecera)
    for (var i = 0; i < cabecera.length; i++) 
      {
        puntos[i] = {label: cabecera[i],data: new Array()};
      } 
    //Parece más optimo, ya que hay que recorrer el archivo de texto linea por linea, aprovechar para 
    //extraer todas las variables del archivo, así luego será más rapido cambiar de la representación
    //de una vble a otra. La alternativa sería tratar el archivo de texto por cada vez que se elija una nueva vble
    lineas.pop();
    var inilineas=4;
    //console.log("lapso: ",lapso)
    if (lineas.length-lapso>4)
        inilineas=lineas.length-lapso;
    for (var i=inilineas;i<lineas.length;i++)
        {
        var fecha=lineas[i].split(",")[0];
        var d=fecha.split(/\-|\s|:|"/);
          d.splice(0, 1); //para eliminar el primer elemento del array, que está vacio
          var xi=Date.UTC(d[0], d[1]-1, d[2], d[3], d[4],d[5]);
      
          //console.log(Voffset);
          //throw new Error("my error message");
          puntos[0].data.push({x: xi, y:xi})
          for (var k=1;k<(cabecera.length);k++)
              {
              var newVal=parseFloat(lineas[i].split(",")[k]);
              var punto={x: xi,y: newVal};
              puntos[k].data.push(punto);
              };
        }
    //datosCargados.value=true
    console.log("PUUUUUUUUNNNNTTTTOOOS: ", puntos)
    }
  function popularVbles(vbles){
    vblesArchivo.value=[]
    for (let i=0;i<vbles.length;i++){
      vblesArchivo.value.push({value:i+1,label:vbles[i]})
    }
  }

  async function fetchListFiles(url) {
  //listadoFicheros.value = null
  const res = await fetch(url)
  //convertimos a json para acceder comodamente a la respuesta recibida:
  const listado=await res.json()
  //extraemos la parte que define el nombre del archivo sin la ruta y con eso
  //populamos el segundo listbox...
  const archivos=listado.map(x=>x.archivo.split('/').pop().split('.')[0])
  archivosProyecto.value=[]
  for (let i=0;i<archivos.length;i++){
    archivosProyecto.value.push({value:i+1,label:archivos[i]})
  }
  //Copiamos el resultado del json a la variable listadoFicheros
  //archivosProyecto.value=[...archivos]
  seleccionadoArchivo.value=true
  //listadoFicheros.value=[...myres]
  
  //Esto muestra en consola el primer valor recibido, osea el primer archivo encontrado en la ruta
  //console.log("res: ",archivosProyecto.value[0])
  //guardamos todo el listado de archivos recibidos como el primer elemento del array listadoFicheros
  //listadoFicheros.value.push(archivos)
  //Para acceder en VUE a una vble tipo ref() hay que usar listadoFicheros.value
  //como en este caso esta vble almacena un array de listados, usamos [0] para acceder al primer listado guardado
  //y para acceder al primer elemento del primer listado: [0][0]
  //console.log(listadoFicheros.value[0][0])
}


  //projectosREC.find(projecto => projecto.value === form.project_id).label
function handleReset(){
  datosCargados.value=false
  console.log("RESET!!")
}
</script>

<template>
  <h1 style="color:aliceblue">Proyectos REC</h1>
  <button v-show="false"
      className="bg-blue-500 text-white text-lg font-medium p-2 rounded-md shadow-lg hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-600 focus:ring-opacity-50 cursor-pointer"
      @click="handle"
    >
      Reset
    </button>

  <div class="p-4 mx-auto max-w-2xl">
    <form
      action="#"
      class="flex flex-col space-y-6"
    >
      <BaseListbox
        placeholder="Selecciona proyecto"
        v-model="form.project_id"
        :options="proyectosREC"
        @response="recibidoProyecto"
      />

      <BaseListbox
        v-if="seleccionadoArchivo"
        placeholder="Selecciona archivos"
        v-model="form.file_id"
        :options="archivosProyecto"
        @response="recibidoArchivo"
      />
      <BaseListbox
        v-if="seleccionadaVble"
        placeholder="Selecciona vable"
        v-model="form.vble_id"
        :options="vblesArchivo"
        @response="recibidaVble"
      />
    </form>
  </div>
  <div style="color:aliceblue" v-if="form.project_id">
    {{  }}
  </div>
  <div v-if="datosCargados">
        <div class="wrapper">
            <grafica :misPuntos="puntosAGraficar"/>
        </div>
    </div>
</template>