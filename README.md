# Reproductor-Mp3

import controlP5.*;
import ddf.minim.*;
import javax.swing.*;
import java.util.*;
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
import java.net.InetAddress;
import javax.swing.filechooser.FileFilter;
import javax.swing.filechooser.FileNameExtensionFilter;
import ddf.minim.analysis.*;
import ddf.minim.effects.*;
import ddf.minim.signals.*;
import ddf.minim.spi.*;
import ddf.minim.ugens.*;
import netP5.*;
import oscP5.*;
static String INDEX_NAME = "canciones";
static String DOC_TYPE = "cancion";

ControlP5 ui, ui2, p5, ecu;
float a[]=new float [100];
Minim minim;
int duracion = 5;
boolean mute = false;
AudioPlayer song;
FFT fft;
boolean c = false;
AudioMetaData meta;

PImage play, pause, stop, abrir, silencio, nosilencio;

ScrollableList list;
Client client;
Node node;



void setup() {

  size(335, 300);
  play = loadImage("playRed.png");
  pause = loadImage("pausaRed.png");
  stop = loadImage("stopRed.png");
  abrir = loadImage("selectSong.png");
  silencio = loadImage("mute.png");
  nosilencio = loadImage("unmute.png");


  Settings.Builder settings = Settings.settingsBuilder();

  settings.put("path.data", "esdata");
  settings.put("path.home", "/");
  settings.put("http.enabled", false);
  settings.put("index.number_of_replicas", 0);
  settings.put("index.number_of_shards", 1);

  node = NodeBuilder.nodeBuilder()
    .settings(settings)
    .clusterName("mycluster")
    .data(true)
    .local(true)
    .node();

  client = node.client();

  ClusterHealthResponse r = client.admin().cluster().prepareHealth().setWaitForGreenStatus().get();

  IndicesExistsResponse ier = client.admin().indices().prepareExists(INDEX_NAME).get();
  if (!ier.isExists()) {
    client.admin().indices().prepareCreate(INDEX_NAME).get();
  }

  ui = new ControlP5(this); 
  ui2 = new ControlP5(this); 
  ui.addButton("Play").setPosition(300, 0).setSize(35, 35).setImage(play);  
  ui.addButton("Pause").setPosition(300, 35 ).setSize(35, 35).setImage(pause);  
  ui.addButton("Stop").setPosition( 300, 70).setSize(35, 35).setImage(stop);  
  ui2.addSlider("Volumen").setPosition( 305, 180).setSize(25, 100).setRange(0, 100).setValue(0);
  ui.addButton("Mute").setPosition(300,105).setSize(35, 35).setImage(nosilencio);
  ui.addButton("importFiles").setPosition(300, 140).setSize(35, 35).setImage(abrir);
  //ui.addKnob("Agudos").setPosition(50, 150).setRange(20, 500).setSize(100, 50);
  //ui.addKnob("Bajos").setPosition(200, 150).setRange(500, 5000).setSize(100, 50);
  //ui.addKnob("Medios").setPosition(350, 150).setRange(5000, 20000).setSize(100, 50);
  p5 = new ControlP5(this);
  list = p5.addScrollableList("playlist").setPosition(0, 0).setSize(300, height).setBarHeight(20).setItemHeight(20).setType(ScrollableList.LIST);


  minim = new Minim(this);
  loadFiles();
}



void draw() {
  background(0);
  fill(#0000ff);
  rect(0, 0, 300, height);
}

public void Play() {
  song.play();
  println("play");
}

public void Pause() {
  song.pause();
  println("pause");
}

public void Stop() {
  //minim.stop();
  song.pause();
  song.rewind();
  println("stop");
  //  cancion();
}

void Volumen(float theColor) {
  float x=0;
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

void Mute() {
  if (!mute) {
    song.mute();
    mute = true;
    nosilencio = loadImage("unmute.png");
  } else {
    song.unmute();
    mute = false;
    nosilencio = loadImage("mute.png");
  }
}

public void importFiles() {
  JFileChooser jfc = new JFileChooser();
  jfc.setFileFilter(new FileNameExtensionFilter("MP3 File", "mp3"));
  jfc.setMultiSelectionEnabled(true);
  jfc.showOpenDialog(null);

  for (File f : jfc.getSelectedFiles()) {
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

    Map<String, Object> doc = new HashMap<String, Object>();
    doc.put("author", meta.author());
    doc.put("title", meta.title());
    doc.put("path", f.getAbsolutePath());

    try {
      client.prepareIndex(INDEX_NAME, DOC_TYPE, f.getAbsolutePath())
        .setSource(doc)
        .execute()
        .actionGet();

      addItem(doc);
    } 
    catch(Exception e) {
      e.printStackTrace();
    }
  }
}

void fileSelected(File selection) {
  if (selection != null) {
    minim.stop();
    song = minim.loadFile(selection.getAbsolutePath(), 1024);
    meta = song.getMetaData();
    c = true;
  } else {
    if (song != null) {
    }
    println("Window was closed or the user hit cancel.");
  }
}


void playlist(int n) {
  Map<String, Object> value = (Map<String, Object>) list.getItem(n).get("value");  
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
}


void loadFiles() {
  try {
    SearchResponse response = client.prepareSearch(INDEX_NAME).execute().actionGet();

    for (SearchHit hit : response.getHits().getHits()) {
      addItem(hit.getSource());
    }
  } 
  catch(Exception e) {
    e.printStackTrace();
  }
}

void addItem(Map<String, Object> doc) {
  list.addItem(doc.get("author") + " - " + doc.get("title"), doc);
}
