---
layout: post 
title: Hierarchical Clustering
categories:
- blog
---

<link rel="stylesheet" type="text/css" href="{{ "/assets/css/jquery-ui-1.8.custom.css" | relative_url }}" />

<div id="Inner">
  <style>
    #Controls {font-size: 14px; }
    .slider {float: left; width:300px; }
    .end-num {float:left; margin-left: 12px; width: 32px;  }
    .slider-row {padding: 5px 0px; }
    #Graphs div {float:left; }
  </style>
  <div id="Controls">
    <div>
    <div class="slider-row" >
      <div id="Slider1" class="slider"></div><div class="end-num" id="k">100</div>
      <div style="clear:both" ></div>
    </div>
    <div>Number of clusters</div>
    </div>
    <div>
    <div class="slider-row" >
      <div id="Slider2" class="slider"></div><div class="end-num" id="p">1</div>
      <div style="clear:both" ></div>
    </div>
    <div>Noise level</div>
    </div>
  </div><!-- Controls -->
  <div id="Graphs">
    <div>
    <canvas id="Graph1" width="300" height="300" ></canvas>
    <br />
    <div>Distances</div>
    </div>
    <div>
    <canvas id="Graph2" width="300" height="300" ></canvas>
    <br />
    <div>Greedy linkage</div>
    </div>
    <div>
    <canvas id="Graph21" width="300" height="300" ></canvas>
    <br />
    <div>Single linkage</div>
    </div>
    <div>
    <canvas id="Graph3" width="300" height="300" ></canvas>
    <br />
    <div>Complete linkage</div>
    </div>
    <div>
    <canvas id="Graph4" width="300" height="300" ></canvas>
    <br />
    <div>Average linkage</div>
    </div>
    <div>
    <canvas id="Graph5" width="300" height="300" ></canvas>
    <br />
    <div>Median linkage</div>
    </div>
    <div>
    <canvas id="Graph6" width="300" height="300" ></canvas>
    <br />
    <div><a href="https://en.wikipedia.org/wiki/Chinese_Whispers_(clustering_method)">Chinese Whispers</a></div>
    </div>
  </div>
<script type="text/javascript" src="{{ "/assets/js/jquery-1.4.2.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/jquery-ui-1.8.custom.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/jquery.ui.touch-punch.min.js" | relative_url }}"></script>
<script type="text/javascript" src="{{ "/assets/js/randomColor.js" | relative_url }}"></script>
<script type="text/javascript" >

var MAX_K = 20;
var N = 100;
var colours = randomColor({count:MAX_K});

jQuery(document).ready(function($){

	/* Generate Nice jQuery UI Sliders */
	$('#Slider1, #Slider2').slider({
		slide: function(event,ui){ doCluster();}
	});

	/* Set some random initial values for the sliders */
  $('#Slider1').slider('value', 25);
  $('#Slider2').slider('value', 0);
  
  $('#Graph1,#Graph2,#Graph21,#Graph3,#Graph4,#Graph5,#Graph6').mouseover(function(){ drawClusters2(JSON.parse($(this).attr('data')), this, true);}).mouseout(function(){ drawClusters2(JSON.parse($(this).attr('data')), this, false);});

	doCluster();
});

var lastK = null;
var lastPoints = null;

var doCluster = function(){
  var k = Math.floor($('#Slider1').slider('value')*MAX_K/100)+1; // range 1-MAX_K
  var p = (Math.round($('#Slider2').slider('value')*20/100)) / 20; // range 0-1
  $('#k').text(k); 
  $('#p').text(p); 

  // generate a random cluster assignment for 100 points
  var points = [];
  if(k==lastK && lastPoints)
    points = lastPoints;
  else {
    for(var i=0;i<N;i++) points[i] = Math.floor(Math.random() * k);
    lastPoints=points;
    lastK=k;
  }
  var distM = [];
  for(var i=0;i<N;i++){
    distM.push([]);
    for(var j=0;j<N;j++){
      distM[i][j] = points[i]==points[j] ? 0 : 1; 
    }
  }
  distM = noise(distM, p);

  // graph distance matrix
  var canvas = document.getElementById("Graph1");
  clearCanvas(canvas);
  var ctx = canvas.getContext("2d");
  var idx = [];
  for(var i=0;i<N;i++) idx[i] = i;
  var sorted = idx.sort(function(a,b){
    if(points[a] < points[b]) return -1;
    else if (points[a] > points[b]) return 1;
    else return 0;
  });
  for(var i=0;i<N;i++){
    for(var j=0;j<N;j++){
      var x = Math.floor(distM[sorted[i]][sorted[j]] * 255);
      ctx.fillStyle = 'rgb('+x+','+x+','+x+')';
      ctx.fillRect(i*3,j*3, 3, 3);
    }
  }
  for(var i=0;i<N;i++){
    ctx.fillStyle=colours[points[sorted[i]]];
    ctx.fillRect(i*3, 0, 3, 2);
    ctx.fillRect(0, i*3, 2, 3);
    ctx.fillRect(i*3, 300-2, 3, 2);
    ctx.fillRect(300-2, i*3, 2, 3);
  }

  // compute greedy-linkage with 0.5 threshold
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    outer:
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] < 0.5) {
              // merge i,j clusters
              clusters[i] = clusters[i].concat(clusters[j])
              clusters.splice(j, 1);
              changed = true;
              break outer;
            }
          }
        }
        if(changed) break;
      }
      if(changed) break;
    }
  }while(changed);
  drawClusters(points, clusters, "Graph2");

  // single-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 0.5;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var min = 1;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] < min)
              min = distM[c1[u]][c2[v]];
          }
        }
        if (min < minDist){
          minDist = min;
          pair = [i,j];
        }
      }
    }
    if(pair!=null){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph21");

  // complete-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var max = 0;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            if(distM[c1[u]][c2[v]] > max)
              max = distM[c1[u]][c2[v]];
          }
        }
        if (max < minDist){
          minDist = max;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph3");

  // average-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var sum = 0;
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            sum += distM[c1[u]][c2[v]]; 
          }
        }
        var avg = sum / (c1.length * c2.length);
        if (avg < minDist){
          minDist = avg;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph4");

  // median-linkage
  var clusters = initClusters(points);
  var changed = false;
  do {
    //console.log(JSON.stringify(clusters));
    changed = false;
    var minDist = 1;
    var pair = null;
    for(var i=0;i<clusters.length;i++){
      for(var j=i+1;j<clusters.length;j++){
        var c1 = clusters[i], c2 = clusters[j];
        compareCluster:
        var dists = []
        for(var u=0;u<c1.length;u++){
          for(var v=0;v<c2.length;v++){
            dists.push(distM[c1[u]][c2[v]]);
          }
        }
        dists = dists.sort();
        var median = dists[Math.floor(dists.length/2)];
        if (median < minDist){
          minDist = median;
          pair = [i,j];
        }
      }
    }
    if(minDist < 0.5){
      var i = pair[0], j = pair[1]; 
      clusters[i] = clusters[i].concat(clusters[j])
      clusters.splice(j, 1);
    } else break;
  }while(true);
  drawClusters(points, clusters, "Graph5");
  // show cluster quality scores for each Graphs

  // Chinese Whispers
  var clusterIdx = [];
  for(var i=0;i<points.length;i++){
    clusterIdx[i] = i; // each point in own cluster 
  }
  var changed = false;
  do {
    //console.log(JSON.stringify(clusterIdx));
    changed = false;
    for(var i=0;i<points.length;i++){             // for each point
      var neighborClusters = []
      for(var j=i+1;j<points.length;j++){       // look at other points
        if(distM[i][j] < 0.5) neighborClusters.push(clusterIdx[j]);
      }
      if (neighborClusters.length!=0){
        var oldCluster = clusterIdx[i]
        clusterIdx[i] = mode(neighborClusters);
        if (clusterIdx[i]!=oldCluster)
          changed = true;
      }
    }
    if (!changed) break; 
  }while(true);
  var id2Cluster = {};
  for(var i=0;i<clusterIdx.length;i++){
    var clId = clusterIdx[i];
    if (id2Cluster[clId]==null) id2Cluster[clId]=[i];
    else id2Cluster[clId].push(i);
  }
  var clusters = [];
  for(key in id2Cluster){
    clusters.push(id2Cluster[key]);
  }
  drawClusters(points, clusters, "Graph6");
}


// adds noise to a 2D matrix
var noise = function(D, p){ 
  for(var i=0;i<N;i++){
    for(var j=0;j<N;j++){
      var x = D[i][j];
      x = nextGaussian(x, p); 
      if(x>1) x=1;
      if(x<0) x=0;
      D[i][j] = x;
    }
  }
  return D;
}

var mode = function(arr){ // return the most common value in arr
  if (arr.length == 0) return null;
  var modeMap = {};
  var maxEl = arr[0], maxCount = 1;
  for(var i=0;i<arr.length;i++){
    var el = arr[i];
    if (modeMap[el]==null) modeMap[el]=1;
    else modeMap[el]++;
    if (modeMap[el] > maxCount){
      maxEl = el;
      maxCount = modeMap[el];
    }
  }  
  return maxEl;
}

var nextGaussian = function(mean,v){
  // polar method of Java Random.nextGaussian()
  var v1=0, v2=0,s=0;
  do {
    v1 = 2 * Math.random() - 1;
    v2 = 2 * Math.random() - 1;
    s = v1 * v1 + v2 * v2;
  }while(s >= 1 || s==0);
  var multiplier = Math.sqrt(-2 * Math.log(s) / s);
  return v1 * multiplier * v + mean;
}

var initClusters = function(points){ // each point is a cluster
  var clusters = [];
  for(var i=0;i<points.length;i++){
    clusters.push([i]);
  }
  return clusters;
}

var clearCanvas = function(element){
  $(element).attr('width',$(element).attr('width'));
}

var drawClusters = function(points, clusters, id, showtext){
  var canvas = document.getElementById(id);
  // stash clusters in the dom
  var cl = [];
  for(var i in clusters){
    var c = []
    for(var j in clusters[i]){
      c.push(points[clusters[i][j]]);
    }
    cl.push(c);
  }
  canvas.setAttribute('data', JSON.stringify(cl));
  drawClusters2(cl, canvas, showtext);
}
var drawClusters2 = function(clusters, canvas, showtext){
  clearCanvas(canvas);
  var ctx = canvas.getContext("2d");
  ctx.font = "bold 16px 'Open Sans', Arial, sans-serif";

  var clusterColorFlat = clusters.reduce(function(a, b) {
  return a.concat(b);
  }, []); 
  var clusterIndexFlat = [];
  var idx =0 ;
  for(var i in clusters){
    for(var j in clusters[i]){
      clusterIndexFlat[idx++] = i;
    }
  }

  var idx=0;
  for(var i=0;i<10;i++){
    for(var j=0;j<10;j++){
      ctx.fillStyle = colours[clusterColorFlat[idx]];
      ctx.fillRect(j*30, i*30, 30, 30);
      ctx.fillStyle = 'white';
      if(showtext){
        ctx.fillText(clusterColorFlat[idx], j*30+8, i*30+20);
      }
      // draw partitions
      var begOfRow = idx % 10 == 0;
      var endOfRow = idx % 10 == 9;
      var topRow = idx < 10;
      var bottomRow = idx >= 90;
      var prevDiff = idx==0 || clusterIndexFlat[idx] != clusterIndexFlat[idx-1];
      var nextDiff = idx==clusterIndexFlat.length-1 || clusterIndexFlat[idx] != clusterIndexFlat[idx+1];
      var topDiff = topRow || clusterIndexFlat[idx] != clusterIndexFlat[idx-10];
      var bottomDiff = bottomRow  || clusterIndexFlat[idx] != clusterIndexFlat[idx+10];
      ctx.fillStyle = '#333';
      if (topRow || topDiff)
        ctx.fillRect(j*30, i*30, 30, 1); //top
      if (bottomRow || bottomDiff) 
        ctx.fillRect(j*30, i*30+29, 30, 1); //bottom
      if(prevDiff || begOfRow )
        ctx.fillRect(j*30, i*30, 1, 30); //left
      if(nextDiff || endOfRow )
        ctx.fillRect(j*30 + 29, i*30, 1, 30); //right
      idx++;
    }
  }
}
</script>
</div>
