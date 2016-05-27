import controlP5.*;
import processing.sound.*;
import ddf.minim.*;
import ddf.minim.ugens.*;
import ddf.minim.effects.*;
import ddf.minim.analysis.FFT;
import ddf.minim.analysis.*;
import oscP5.*;
import netP5.*;

import org.elasticsearch.action.admin.indices.exists.indices.IndicesExistsResponse;
import org.elasticsearch.action.admin.cluster.health.ClusterHealthResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.Client;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.node.Node;
import org.elasticsearch.node.NodeBuilder;
import java.util.*;
import java.net.InetAddress;
import javax.swing.*;
import javax.swing.filechooser.FileFilter;
import javax.swing.filechooser.FileNameExtensionFilter;

static String INDEX_NAME = "canciones";
static String DOC_TYPE = "cancion";
ScrollableList list;

Client client;
Node node;


PImage img, img2, img3, img4;
ControlP5 pl, stop, pa, ei, k, j, cp5;
ControlP5 ui;
int segundos=0, minutos=0, ta=0, casiseg = 0;
boolean m=false, press= false, seleccionados;
int agu = 10;
float v = 0;
AudioPlayer  songa;
boolean mute = false;
float a[]=new float [100];
Gain gain;
FilePlayer filePlayer;
Minim minim;
AudioPlayer song;
AudioMetaData meta;
FFT fft;
boolean c=false;
void setup() {
  size(1001, 601 );
  background(#000020);
  
  //lista
  // Configuracion basica para ElasticSearch en local
  Settings.Builder settings = Settings.settingsBuilder();
  // Esta carpeta se encontrara dentro de la carpeta del Processing
  settings.put("path.data", "esdata");
  settings.put("path.home", "/");
  settings.put("http.enabled", false);
  settings.put("index.number_of_replicas", 0);
  settings.put("index.number_of_shards", 1);  
  // Inicializacion del nodo de ElasticSearch
  node = NodeBuilder.nodeBuilder()
    .settings(settings)
    .clusterName("mycluster")
    .data(true)
    .local(true)
    .node();
  // Instancia de cliente de conexion al nodo de ElasticSearch
  client = node.client();

  // Esperamos a que el nodo este correctamente inicializado
  ClusterHealthResponse r = client.admin().cluster().prepareHealth().setWaitForGreenStatus().get();
  println(r);

  // Revisamos que nuestro indice (base de datos) exista
  IndicesExistsResponse ier = client.admin().indices().prepareExists(INDEX_NAME).get();
  if (!ier.isExists()) {
    // En caso contrario, se crea el indice
    client.admin().indices().prepareCreate(INDEX_NAME).get();
  }

     

  minim= new Minim(this);


  img=loadImage("cosas.jpg");
  image(img, 0, 0);
  pl = new ControlP5(this);
  pl.addButton("Play").setPosition(280, 300).setImages(loadImage("play.png"), loadImage("play.png"), loadImage("play.png")).updateSize();
  stop=new ControlP5(this);
  stop.addButton("Detener").setPosition(575, 430).setImages(loadImage("stop.png"), loadImage("stop.png"), loadImage("stop.png")).updateSize(); 
  pa=new ControlP5(this);
  pa.addButton("Parar").setPosition(220, 430).setImages(loadImage("pause.png"), loadImage("pause.png"), loadImage("pause.png")).updateSize();
  pa=new ControlP5(this);
  pa.addButton("Mute").setPosition(10, 530).setSize(50, 30).setImages(loadImage("images.png"), loadImage("images.png"), loadImage("images.png")).updateSize();
  pa=new ControlP5(this);
  pa.addButton("seleccionar").setPosition(20, 10).setSize(70, 30).setColorValue(color(255)).setColorBackground(color(0));
  pa=new ControlP5(this);
  gain = new Gain(0.f) ;
  pa.addSlider("Volumen").setPosition( 810, 480).setSize(10, 100).setRange(0, 100).setValue(0).setNumberOfTickMarks(30).setColorValue(color(255)).setColorBackground(color(0, 0, 255));
  pa.getController("Volumen").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(1000);
  minim= new Minim(this);
  song=minim.loadFile("stream.mp3", 1024);

  ei = new ControlP5(this);
  ei.addSlider("Hpass").setPosition(850, 480).setSize(10, 100).setRange(0, 3000).setValue(0).setNumberOfTickMarks(30).setColorValue(color(255)).setColorBackground(color(0, 0, 255));
  ei.addSlider("Lpass").setPosition(895, 480).setSize(10, 100).setRange(3000, 20000).setValue(3000).setNumberOfTickMarks(30).setColorValue(color(255)).setColorBackground(color(0, 0, 255));
  ei.addSlider("Bpass").setPosition(935, 480).setSize(10, 100).setRange(100, 1000).setValue(100).setNumberOfTickMarks(30).setColorValue(color(255)).setColorBackground(color(0, 0, 255));
  ei.getController("Hpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(1000);
  ei.getController("Lpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(1000);
  ei.getController("Bpass").getValueLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingY(1000);

  j = new ControlP5(this);
  j.addSlider("tiempo").setPosition(200, 350).setRange(0, agu).setSize(520, 10);


   // Agregamos a la vista un boton de importacion de archivos
 cp5 = new ControlP5(this);
 cp5.addButton("importFiles")
   .setPosition(700, 0).setSize(100,20)
   .setLabel("Importar archivos");

  // Agregamos a la vista una lista scrollable que mostrara las canciones
  list = cp5.addScrollableList("playlist")
   .setPosition(720, 25)
   .setSize(280, 400)
   .setBarHeight(20)
   .setItemHeight(20)
   .setType(ScrollableList.LIST);

  // Cargamos los archivos de la base de datos
  loadFiles();
}

void draw() {
  if (seleccionados) {
    visAudio();
    img2=loadImage("cosas.jpg");
    background(img2);
    j.getController("tiempo").setValue(song.position());
    segundos = song.position()/1000%60;
    minutos = song.position()/(60*1000)%60;
    String o = nf(segundos, 2);
    String p = nf(minutos, 2);
    text(p+":"+o, 165, 380); 
    int segd = agu/1000%60;
    int mind = agu/(60*1000)%60;
    String q = nf(segd, 2);
    String r= nf(mind, 2);

    visAudio();
    img4=loadImage("cosas3.jpg");
    image(img4, 0, 0);

    img3=loadImage("cosas2.jpg");
    image(img3, 720, 0);


    text("Duracion: "+r+":"+ q, 10, 120);
    casiseg = song.position();
    ta = agu - song.position();
    int s = ta/1000%60;
    int t = ta/(60*1000)%60;
    String u = nf(s, 2);
    String v = nf(t, 2);
    text("-"+v+":"+u, 680, 380);
    textSize(20);
    text("Titulo: " + meta.title(), 10, 65);
    text("Autor: " + meta.author(), 10, 90);
  }
  
  fill(0);
  textSize(20);
  text("Ecualizador", 825, 450);
  }
  
  public void Play() {
  song.play();
  println("Play");
}
public void Detener() {
  song.pause();
  song.rewind();
  println("Stop");
}
public void Parar() {
  song.pause();
  println("Pause");
}

void Volumen(float theColor) {
  float mycolor=theColor;

  for (int i=0; i<30; i++) {
    a[i]=theColor;
    if (a[i+1]<a[i]) {
      a[i+1]=song.getGain()+ theColor;
      song.setGain( song.getGain()+mycolor);
    } else if (  a[i+1]>a[i]) {
      a[i+1]=song.getGain()- theColor;
      song.setGain( song.getGain()- mycolor);
    }
  }
}

void Mute () {
  if (!mute) {
    song.mute();
    mute = true;
  } else {
    song.unmute();
    mute = false;
  }
}
void seleccionar() {
  selectInput("Select a file to process:", "fileSelected");
}

void fileSelected(File selection) {
  if (selection != null) {
    
    seleccionados = true;
    minim.stop();
    song = minim.loadFile(selection.getAbsolutePath(), 1024);
    meta = song.getMetaData();
     c=true;
    fft = new FFT(song.bufferSize(), song.sampleRate());

    agu = song.length();

    j = new ControlP5(this);
    j.addSlider("tiempo").setPosition(200, 350).setRange(0, agu).setSize(520, 10);
    j.getController("tiempo").getValueLabel().align(ControlP5.LEFT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);
    j.getController("tiempo").getCaptionLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);

    seleccionados = true;
  } else {
    seleccionados = false;
    if (song != null) {
      seleccionados = true;
    }
    println("Window was closed or the user hit cancel. ");
  }
}

void loadSong() {
  song = minim.loadFile(songa+"");
}

void visAudio() {
 
  for (int i = 0; i < song.bufferSize()-1; i++)
  {
    stroke(0);
    line(i, 300+song.left.get(i)*50, i+1, 300 +song.left.get(i+1)*50 );
    line(i, 150+song.left.get(i)*50, i+1, 300 +song.left.get(i+1)*50 );
    line(i, 250 + song.right.get(i)*50, i+1, 250 + song.right.get(i+1)*50);
    line(i, 250 + song.right.get(i)*50, i+1, 250 + song.right.get(i+1)*50);
  }
  for (int i = 0; i < song.bufferSize()-1; i++)
  {
    stroke(#0000CC);
    line(i, 300+song.left.get(i)*50, i+1, 300 +song.left.get(i+1)*50 );
    line(i, 300+song.left.get(i)*50, i+1, 300 +song.left.get(i+1)*50 );
    line(i, 200 + song.right.get(i)*50, i+1, 200 + song.right.get(i+1)*50);
    line(i, 300 + song.right.get(i)*50, i+1, 300 + song.right.get(i+1)*50);
    line(i, 250 + song.right.get(i)*50, i+1, 250 + song.right.get(i+1)*50);
    line(i, 150 + song.right.get(i)*50, i+1, 150 + song.right.get(i+1)*50);
  }
  for (int i = 0; i < song.bufferSize()-1; i++)
  {
    stroke(#0066CC);
    line(i, 200 + song.right.get(i)*50, i+1, 250 + song.right.get(i+1)*50);
    line(i, 250 + song.right.get(i)*50, i+1, 200 + song.right.get(i+1)*50);
  }
  for (int i = 0; i < song.bufferSize()-1; i++)
  {
    stroke(255);
    line(i, 250 + song.right.get(i)*50, i+1, 250 + song.right.get(i+1)*50);
    line(i, 290 + song.right.get(i)*50, i+1, 290 + song.right.get(i+1)*50);
    //line(i, 210 + song.right.get(i)*50, i+1, 210 + song.right.get(i+1)*50);
    line(i, 170 + song.right.get(i)*50, i+1, 170 + song.right.get(i+1)*50);
    //line(i, 230 + song.right.get(i)*50, i+1, 230 + song.right.get(i+1)*50);
    line(i, 190 + song.right.get(i)*50, i+1, 190 + song.right.get(i+1)*50);
  }
  for (int i = 0; i < song.bufferSize()-1; i++)
  {
    stroke(0);
    line(i, 210 + song.right.get(i)*50, i+1, 210 + song.right.get(i+1)*50);
    line(i, 230 + song.right.get(i)*50, i+1, 230 + song.right.get(i+1)*50);
  }
}

//listas

public void importFiles() {
   j = new ControlP5(this);
    j.addSlider("tiempo").setPosition(200, 350).setRange(0, agu).setSize(520, 10);
    j.getController("tiempo").getValueLabel().align(ControlP5.LEFT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);
    j.getController("tiempo").getCaptionLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);

    seleccionados = true;
  
  // Selector de archivos
  JFileChooser jfc = new JFileChooser();
  // Agregamos filtro para seleccionar solo archivos .mp3
  jfc.setFileFilter(new FileNameExtensionFilter("MP3 File", "mp3"));
  // Se permite seleccionar multiples archivos a la vez
  jfc.setMultiSelectionEnabled(true);
  // Abre el dialogo de seleccion
  jfc.showOpenDialog(null);

  // Iteramos los archivos seleccionados
  for (File f : jfc.getSelectedFiles()) {
    // Si el archivo ya existe en el indice, se ignora
    GetResponse response = client.prepareGet(INDEX_NAME, DOC_TYPE, f.getAbsolutePath()).setRefresh(true).execute().actionGet();

    if (minim != null) {
      minim.stop();
      minim = new Minim(this);
      song = minim.loadFile(f.getAbsolutePath());
      meta = song.getMetaData();
    } else {
      minim = new Minim(this);
      song = minim.loadFile(f.getAbsolutePath());
      meta = song.getMetaData();
    }

    if (response.isExists()) {
      continue;
    }



    //// Cargamos el archivo en la libreria minim para extrar los metadatos
    //Minim minim = new Minim(this);
    //AudioPlayer song = minim.loadFile(f.getAbsolutePath());
    //AudioMetaData meta = song.getMetaData();

    // Almacenamos los metadatos en un hashmap
    Map<String, Object> doc = new HashMap<String, Object>();
    doc.put("author", meta.author());
    doc.put("title", meta.title());
    doc.put("path", f.getAbsolutePath());

    try {
      // Le decimos a ElasticSearch que guarde e indexe el objeto
      client.prepareIndex(INDEX_NAME, DOC_TYPE, f.getAbsolutePath())
        .setSource(doc)
        .execute()
        .actionGet();

      // Agregamos el archivo a la lista
      addItem(doc);
    } 
    catch(Exception e) {
      e.printStackTrace();
    }
  }
}
// Al hacer click en algun elemento de la lista, se ejecuta este metodo
void playlist(int n) {
  Map value = (Map) list.getItem(n).get("value");
  if (minim != null) { 
    minim.stop(); 
    minim = new Minim(this); 
    song = minim.loadFile((String)value.get("path")); 
    meta = song.getMetaData(); 
    c = true;
  } else { 
    minim = new Minim(this); 
    song = minim.loadFile((String)value.get("path")); 
    meta = song.getMetaData(); 
    c = true;
  }
  println(list.getItem(n));
   fft = new FFT(song.bufferSize(), song.sampleRate());
   
     j = new ControlP5(this);
    j.addSlider("tiempo").setPosition(200, 350).setRange(0, agu).setSize(520, 10);
    j.getController("tiempo").getValueLabel().align(ControlP5.LEFT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);
    j.getController("tiempo").getCaptionLabel().align(ControlP5.RIGHT, ControlP5.BOTTOM_OUTSIDE).setPaddingX(0).setPaddingY(30);

    seleccionados = true;
}

void loadFiles() {
  try {
    // Buscamos todos los documentos en el indice
    SearchResponse response = client.prepareSearch(INDEX_NAME).execute().actionGet();

    // Se itera los resultados
    for (SearchHit hit : response.getHits().getHits()) {
      // Cada resultado lo agregamos a la lista
      addItem(hit.getSource());
    }
  } 
  catch(Exception e) {
    e.printStackTrace();
  }
  
}
//void addItem (Map doc) { list.addItem(doc.get("author") + " - " + doc.get("title"), doc); }

// Metodo auxiliar para no repetir codigo
void addItem(Map doc) {
  // Se agrega a la lista. El primer argumento es el texto a desplegar en la lista, el segundo es el objeto que queremos que almacene
  list.addItem(doc.get("author") + " - " + doc.get("title"), doc);
}

  

