<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.0 Transitional//EN">
<HTML>

<HEAD>
  <TITLE> YÊU HOA </TITLE>
  <META NAME="Generator" CONTENT="EditPlus">
  <META NAME="Author" CONTENT="">
  <META NAME="Keywords" CONTENT="">
  <META NAME="Description" CONTENT="">


  <style>
    html,
    body {
      height: 100%;
      padding: 0;
      margin: 0;
      background: #000;
      display: flex;
      justify-content: center;
      align-items: center;

    }

    .box {
      width: 100%;
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      display: flex;
      flex-direction: column;
    }

    canvas {
      position: absolute;
      width: 100%;
      height: 100%;
    }

    #pinkboard {
      position: relative;
      margin: auto;
      height: 500px;
      width: 500px;
      animation: animate 1.3s infinite;
    }

    #pinkboard:before,
    #pinkboard:after {
      content: '';
      position: absolute;
      background: red;
      width: 100px;
      height: 160px;
      border-top-left-radius: 50px;
      border-top-right-radius: 50px;
    }

    #pinkboard:before {
      left: 100px;
      transform: rotate(-45deg);
      transform-origin: 0 100%;
      box-shadow: 0 14px 28px rgba(0, 0, 0, 0.25),
        0 10px 10px rgba(0, 0, 0, 0.22);
    }

    #pinkboard:after {
      left: 0;
      transform: rotate(45deg);
      transform-origin: 100% 100%;
    }

    @keyframes animate {
      0% {
        transform: scale(1);
      }

      30% {
        transform: scale(.8);
      }

      60% {
        transform: scale(1.2);
      }

      100% {
        transform: scale(1);
      }
    }
  </style>
</HEAD>

<BODY>
  <div class="box">
    <canvas id="pinkboard"></canvas>
  </div>
  <script>
    /*
   * Settings
   */
    
    var settings = {
      pas: {
        length: 2000, // maximum amount of pas
        duration: 2, // pa duration in sec
        velocity: 100, // pa velocity in pixels/sec
        effect: -1, // play with this for a nice effect
        size: 10, // pa size in pixels
      },
    };
    /*
     * RequestAnimationFrame polyfill by Erik Möller
     */
    (function () {var b = 0; var c = ["ms", "moz", "webkit", "o"]; for (var a = 0; a < c.length && !window.requestAnimationFrame; ++a) {window.requestAnimationFrame = window[c[a] + "RequestAnimationFrame"]; window.cancelAnimationFrame = window[c[a] + "CancelAnimationFrame"] || window[c[a] + "CancelRequestAnimationFrame"]} if (!window.requestAnimationFrame) {window.requestAnimationFrame = function (h, e) {var d = new Date().getTime(); var f = Math.max(0, 16 - (d - b)); var g = window.setTimeout(function () {h(d + f)}, f); b = d + f; return g}} if (!window.cancelAnimationFrame) {window.cancelAnimationFrame = function (d) {clearTimeout(d)}} }());
    /*
     * Point class
     */
    var Point = (function () {
      function Point(x, y) {
        this.x = (typeof x !== 'undefined') ? x : 0;
        this.y = (typeof y !== 'undefined') ? y : 0;
      }
      Point.prototype.clone = function () {
        return new Point(this.x, this.y);
      };
      Point.prototype.length = function (length) {
        if (typeof length == 'undefined')
          return Math.sqrt(this.x * this.x + this.y * this.y);
        this.normalize();
        this.x *= length;
        this.y *= length;
        return this;
      };
      Point.prototype.normalize = function () {
        var length = this.length();
        this.x /= length;
        this.y /= length;
        return this;
      };
      return Point;
    })();
    /*
     * Pa class
     */
    var Pa = (function () {
      function Pa() {
        this.position = new Point();
        this.velocity = new Point();
        this.acceleration = new Point();
        this.age = 0;
      }
      Pa.prototype.initialize = function (x, y, dx, dy) {
        this.position.x = x;
        this.position.y = y;
        this.velocity.x = dx;
        this.velocity.y = dy;
        this.acceleration.x = dx * settings.pas.effect;
        this.acceleration.y = dy * settings.pas.effect;
        this.age = 0;
      };
      Pa.prototype.update = function (deltaTime) {
        this.position.x += this.velocity.x * deltaTime;
        this.position.y += this.velocity.y * deltaTime;
        this.velocity.x += this.acceleration.x * deltaTime;
        this.velocity.y += this.acceleration.y * deltaTime;
        this.age += deltaTime;
      };
      Pa.prototype.draw = function (context, image) {
        function ease(t) {
          return (--t) * t * t + 1;
        }
        var size = image.width * ease(this.age / settings.pas.duration);
        context.globalAlpha = 1 - this.age / settings.pas.duration;
        context.drawImage(image, this.position.x - size / 2, this.position.y - size / 2, size, size);
      };
      return Pa;
    })();
    var PaPool = (function () {
      var pas,
        firstActive = 0,
        firstFree = 0,
        duration = settings.pas.duration;

      function PaPool(length) {
        // create and populate pa pool
        pas = new Array(length);
        for (var i = 0; i < pas.length; i++)
          pas[i] = new Pa();
      }
      PaPool.prototype.add = function (x, y, dx, dy) {
        pas[firstFree].initialize(x, y, dx, dy);

        // handle circular queue
        firstFree++;
        if (firstFree == pas.length) firstFree = 0;
        if (firstActive == firstFree) firstActive++;
        if (firstActive == pas.length) firstActive = 0;
      };
      PaPool.prototype.update = function (deltaTime) {
        var i;

        // update active pas
        if (firstActive < firstFree) {
          for (i = firstActive; i < firstFree; i++)
            pas[i].update(deltaTime);
        }
        if (firstFree < firstActive) {
          for (i = firstActive; i < pas.length; i++)
            pas[i].update(deltaTime);
          for (i = 0; i < firstFree; i++)
            pas[i].update(deltaTime);
        }

        // remove inactive pas
        while (pas[firstActive].age >= duration && firstActive != firstFree) {
          firstActive++;
          if (firstActive == pas.length) firstActive = 0;
        }


      };
      PaPool.prototype.draw = function (context, image) {
        // draw active pas
        if (firstActive < firstFree) {
          for (i = firstActive; i < firstFree; i++)
            pas[i].draw(context, image);
        }
        if (firstFree < firstActive) {
          for (i = firstActive; i < pas.length; i++)
            pas[i].draw(context, image);
          for (i = 0; i < firstFree; i++)
            pas[i].draw(context, image);
        }
      };
      return PaPool;
    })();
    /*
     * Putting it all together
     */
    (function (canvas) {
      var context = canvas.getContext('2d'),
        pas = new PaPool(settings.pas.length),
        paRate = settings.pas.length / settings.pas.duration, // pas/sec
        time;

      // get point on heart with -PI <= t <= PI
      function pointOnHeart(t) {
        return new Point(
          160 * Math.pow(Math.sin(t), 3),
          130 * Math.cos(t) - 50 * Math.cos(2 * t) - 20 * Math.cos(3 * t) - 10 * Math.cos(4 * t) + 25
        );
      }

      var image = (function () {
        var canvas = document.createElement('canvas'),
          context = canvas.getContext('2d');
        canvas.width = settings.pas.size;
        canvas.height = settings.pas.size;

        function to(t) {
          var point = pointOnHeart(t);
          point.x = settings.pas.size / 2 + point.x * settings.pas.size / 350;
          point.y = settings.pas.size / 2 - point.y * settings.pas.size / 350;
          return point;
        }

        context.beginPath();
        var t = -Math.PI;
        var point = to(t);
        context.moveTo(point.x, point.y);
        while (t < Math.PI) {
          t += 0.01; 
          point = to(t);
          context.lineTo(point.x, point.y);
        }
        context.closePath();

        context.fillStyle = '#e81017';
        context.fill();
        var image = new Image();
        image.src = canvas.toDataURL();
        return image;
      })();

      // render that thing!
      function render() {
        // next animation frame
        requestAnimationFrame(render);

        // update time
        var newTime = new Date().getTime() / 1000,
          deltaTime = newTime - (time || newTime);
        time = newTime;
        // clear canvas
        context.clearRect(0, 0, canvas.width, canvas.height);

        // create new pas
        var amount = paRate * deltaTime;
        for (var i = 0; i < amount; i++) {
          var pos = pointOnHeart(Math.PI - 2 * Math.PI * Math.random());
          var dir = pos.clone().length(settings.pas.velocity);
          pas.add(canvas.width / 2 + pos.x, canvas.height / 2 - pos.y, dir.x, -dir.y);
        }

        // update and draw pas
        pas.update(deltaTime);
        pas.draw(context, image);
      }

      // handle (re-)sizing of the canvas
      function onResize() {
        canvas.width = canvas.clientWidth;
        canvas.height = canvas.clientHeight;
      }
      window.onresize = onResize;

      // delay rendering bootstrap
      setTimeout(function () {
        onResize();
        render();
      }, 10);
    
    })(document.getElementById('pinkboard'));
  </script>
  <div class ="text", style="font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  width: 20px;
  height: 20px;
  background-color:rgba(0, 0, 0, 0.22);
  color: red;
  font-size: 40px;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 5px;
  text-align: center;
  animation: animate 1.3s infinite;">
  HOA 
  </div>
</BODY>

</HTML>
