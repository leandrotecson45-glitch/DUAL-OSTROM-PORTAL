
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DS26_OW Dual OCR → EXIF PRO</title>

<script src="https://cdn.jsdelivr.net/npm/tesseract.js@4/dist/tesseract.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/piexifjs@1.0.6/piexif.min.js"></script>

<style>
body{
  margin:0;
  font-family:Segoe UI;
  background:linear-gradient(rgba(0,0,0,0.6),rgba(0,0,0,0.6)),
  url('https://images.unsplash.com/photo-1502082553048-f009c37129b9') center/cover fixed;
  color:#333;
}

.container{
  max-width:1000px;
  margin:20px auto;
  background:white;
  border-radius:20px;
  padding:15px;
}

/* HEADER WITH PHOTO */
.header{
  width:100%;
  height:180px;
  border-radius:15px;
  overflow:hidden;
  margin-bottom:15px;
  position:relative;
  background: url('https://images.unsplash.com/photo-1492724441997-5dc865305da7') center/cover;
}

.header .overlay{
  width:100%;
  height:100%;
  background: rgba(0,0,0,0.5);
  display:flex;
  flex-direction:column;
  justify-content:center;
  align-items:center;
  color:white;
  text-align:center;
}

.header .overlay h1{
  margin:0;
  font-size:22px;
}

.header .overlay p{
  margin-top:5px;
  font-size:14px;
  opacity:0.9;
}

.grid{
  display:grid;
  grid-template-columns:1fr 1fr;
  gap:10px;
}

.card{
  background:#f9f9f9;
  border-radius:15px;
  padding:10px;
}

img{
  width:100%;
  border-radius:10px;
}

input,textarea{
  width:100%;
  margin-top:6px;
  padding:8px;
  border-radius:8px;
  border:1px solid #ccc;
  font-size:13px;
}

button{
  width:100%;
  margin-top:8px;
  padding:10px;
  border:none;
  background:#00796b;
  color:white;
  border-radius:10px;
}

@media(max-width:700px){
  .grid{grid-template-columns:1fr;}
}
</style>
</head>

<body>

<div class="container">

  <!-- HEADER PHOTO -->
  <div class="header">
    <div class="overlay">
      <h1>📸 DS26_OW Portal</h1>
      <p>Dual OCR → EXIF Auto Processing</p>
    </div>
  </div>

  <div class="grid">
    <div class="card" id="slot1"></div>
    <div class="card" id="slot2"></div>
  </div>
</div>

<script>
function createSlot(containerId){

  let container=document.getElementById(containerId);

  container.innerHTML=`
    <input type="file">
    <img>
    <textarea placeholder="OCR Output"></textarea>

    <input placeholder="Filename">
    <input placeholder="Description">
    <input value="Timestamp Camera" readonly>

    <input placeholder="Model">
    <input placeholder="Make">

    <input placeholder="Latitude">
    <input placeholder="Longitude">

    <button>Download</button>
  `;

  let [file, img, ocr, filename, desc, software, model, make, lat, lon, btn] = container.children;
  let original;

  function extractCoordinates(text){
    text=text.replace(/O/g,'0').replace(/o/g,'0')
             .replace(/l/g,'1').replace(/I/g,'1');

    let nums=text.match(/-?\d{1,3}\.\d+/g);
    if(!nums||nums.length<2)return null;

    nums=nums.map(n=>parseFloat(n));
    let lat=null, lon=null;

    nums.forEach(n=>{
      if(n>=4 && n<=25 && lat===null) lat=n;
      else if(n>=115 && n<=130 && lon===null) lon=n;
    });

    if(lat===null||lon===null){lat=nums[0]; lon=nums[1];}
    if(Math.abs(lat)>90){let t=lat; lat=lon; lon=t;}
    return {lat,lon};
  }

  file.onchange=e=>{
    let f=e.target.files[0];
    if(!f)return;

    filename.value="DS26_OW-"+f.name;
    desc.value="DS26_OW-"+f.name;

    let reader=new FileReader();
    reader.onload=()=>{
      original=reader.result;
      img.src=original;

      Tesseract.recognize(original,'eng').then(({data:{text}})=>{
        ocr.value=text;

        let coords=extractCoordinates(text);
        if(coords){
          lat.value=coords.lat.toFixed(6);
          lon.value=coords.lon.toFixed(6);
        }
      });
    };
    reader.readAsDataURL(f);
  };

  function decimalToDMSRational(dec){
    let abs=Math.abs(dec);
    let deg=Math.floor(abs);
    let minFloat=(abs-deg)*60;
    let min=Math.floor(minFloat);
    let sec=(minFloat-min)*60;
    return [[deg,1],[min,1],[Math.round(sec*100),100]];
  }

  btn.onclick=()=>{
    let exifObj={"0th":{},"GPS":{}};

    exifObj["0th"][piexif.ImageIFD.ImageDescription]=desc.value;
    exifObj["0th"][piexif.ImageIFD.Software]=software.value;
    exifObj["0th"][piexif.ImageIFD.Model]=model.value;
    exifObj["0th"][piexif.ImageIFD.Make]=make.value;

    let la=parseFloat(lat.value);
    let lo=parseFloat(lon.value);

    if(!isNaN(la)&&!isNaN(lo)){
      exifObj["GPS"][piexif.GPSIFD.GPSLatitude]=decimalToDMSRational(la);
      exifObj["GPS"][piexif.GPSIFD.GPSLatitudeRef]=la>=0?"N":"S";

      exifObj["GPS"][piexif.GPSIFD.GPSLongitude]=decimalToDMSRational(lo);
      exifObj["GPS"][piexif.GPSIFD.GPSLongitudeRef]=lo>=0?"E":"W";
    }

    let exifBytes=piexif.dump(exifObj);
    let newImg=piexif.insert(exifBytes,original);

    let a=document.createElement("a");
    let fname=filename.value;
    if(!fname.endsWith(".jpg")) fname+=".jpg";

    a.href=newImg;
    a.download=fname;
    a.click();
  };
}

createSlot("slot1");
createSlot("slot2");
</script>

</body>
</html>
