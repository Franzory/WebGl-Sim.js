# WebGl-Sim.js
[javascript] view plain copy
// Sim.js - A Simple Simulator for WebGL (based on Three.js)  
  
Sim = {};//空对象，用来构建后续的框架  
  
// Sim.Publisher - base class for event publishers  
// 以下为一个订阅者模式的基类，也称为观察者模式，是几乎sim.js对象的基类  
Sim.Publisher = function() {//Publisher的构造函数  
    this.messageTypes = {};//构造函数的属性  
}  
  
Sim.Publisher.prototype.subscribe = function(message, subscriber, callback) {//Publisher的原型方法  
    var subscribers = this.messageTypes[message];  
    if (subscribers)  
    {  
        if (this.findSubscriber(subscribers, subscriber) != -1)//若是在数组里面找到了，说明之前注册了，直接退出  
        {  
            return;  
        }  
    }  
    else  
    {  
        subscribers = [];  
        this.messageTypes[message] = subscribers;  
    }  
  
    subscribers.push({ subscriber : subscriber, callback : callback });//这句才是这个函数的关键，将订阅的函数  
                                                                       //以及回调函数压入subscribers这个栈  
}  
  
Sim.Publisher.prototype.unsubscribe =  function(message, subscriber, callback) {  
    if (subscriber)  
    {  
        var subscribers = this.messageTypes[message];  
  
        if (subscribers)  
        {  
            var i = this.findSubscriber(subscribers, subscriber, callback);//为何这里是三个参数   
                                                                           //下面定义的明明是两个参数  
                                                                           //js对传入的参数无所谓？  
                                                                           //解决：的确无所谓，js都传入的参数个数无所谓  
                                                                           //对于多了的参数自动忽略，但不能使未定义的参数  
                                                                           //对于少传了参数的函数也能运行，只是这个参数默认为未定义  
            if (i != -1)  
            {  
                this.messageTypes[message].splice(i, 1);//从数组中删除该  
            }  
        }  
    }  
    else  
    {  
        delete this.messageTypes[message];  
    }  
}  
  
Sim.Publisher.prototype.publish = function(message) {  
    var subscribers = this.messageTypes[message];  
  
    if (subscribers)  
    {  
        for (var i = 0; i < subscribers.length; i++)  
        {  
            var args = [];   
            for (var j = 0; j < arguments.length - 1; j++)//这里的arguments到底是从哪里来的argument属于谁？  
                                                          //为什么不直接传入到下面的apply中？作用域问题？  
            {  
                args.push(arguments[j + 1]);//这里为什么要+1？？  
            }  
            //  
            subscribers[i].callback.apply(subscribers[i].subscriber, args);//什么意思？  
             //subcribers里面的回调函数被执行同时传入参数  ，apply是调用函数 第一个参数为callback的作用域  
             //第二个是传入的参数数组（js的参数列表本身就是一个“类似数组”）  
        }  
    }  
}  
  
Sim.Publisher.prototype.findSubscriber = function (subscribers, subscriber) {//找到指定索引的位置  
    for (var i = 0; i < subscribers.length; i++)  
    {  
        if (subscribers[i] == subscriber)  
        {  
            return i;  
        }  
    }  
      
    return -1;  
}  
  
  
  
  
  
  
  
  
// Sim.App - application class (singleton)这里是个单体模式？？?????不懂  
//这里不是标准的单体设计模式   
Sim.App = function()  
{  
    Sim.Publisher.call(this);  
      
    this.renderer = null;   //入口在Sim.App.prototype.init  
    this.scene = null;      //入口在Sim.App.prototype.init  
    this.camera = null;     //入口在Sim.App.prototype.init  
    this.objects = [];      //入口在Sim.App.prototype.addObject  
}  
  
Sim.App.prototype = new Sim.Publisher;//这里和前面的call一起，采用的是组合继承模式，(还有一种是采用three.js  
                                      //的方法，Object.create();)但是这里的new会出现两个问题，  
                                      //1.call已经继承了实例对象的属性，但是这里Sim.App.prototype不但继承了原型对象对象  
                                      //还再次继承了实例对象  
                                      //2.这里的Sim.App的construtor直接指向了Sim.Publisher 会造成原型链紊乱  
                                      //解决方案：使用寄生组合继承. 详见javascript高级程序设计E3 6.3.6 以及阮一峰博客http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html  
  
  
  
Sim.App.prototype.init = function(param)//初始化，this会被绑定  
{  
    param = param || {};      
    var container = param.container;  
    var canvas = param.canvas;  
      
    // Create the Three.js renderer, add it to our div  
    var renderer = new THREE.WebGLRenderer( { antialias: true, canvas: canvas } );  
    renderer.setSize(container.offsetWidth, container.offsetHeight);  
    container.appendChild( renderer.domElement );  
  
    // Create a new Three.js scene  
    var scene = new THREE.Scene();  
    scene.add( new THREE.AmbientLight( 0x505050 ) );  
    scene.data = this;  
  
    // Put in a camera at a good default location  
    camera = new THREE.PerspectiveCamera( 45, container.offsetWidth / container.offsetHeight, 1, 10000 );  
    camera.position.set( 0, 0, 3.3333 );  
  
    scene.add(camera);  
      
    // Create a root object to contain all other scene objects  
    var root = new THREE.Object3D();  
    scene.add(root);  
      
    // Create a projector to handle picking  
    var projector = new THREE.Projector();  
      
  
  
  
    // Save away a few things那么问题来了，这里的this是指的哪里的this？？？？作用域是整个Sim.App  
    //还是Sim.App下面的 ???  
    //问题解决：首先是寻找自己作用域（原型函数）下面的，然后是自己所在构造函数下面的  
    //然后再是父类，依次寻找。  
    //对于其他原型函数里面的，只有绑定到了实例对象 里面以后才行也就是说那个原型函数必须调用后   
    //里面的this.XX 属性才能被其他prototype 调用  
/* 
如下图所示 ** 所在位置 若出现this.**  其依次搜寻位置优先级依次为为12345 所用继承方式为 为寄生组合继承 
 
           |super| 
            / \ 
           / | \ 
    |5构造|  |  |3原型| 
             | 
           |sub| 
          /  |  \ 
         /   |   \ 
  |4构造|    |    |1 原型a 属性**| 
             | 
           |2原型b| 
*/  
//由于原型函数下的init（）会被调用用来初始化，所以以下属性在APP全局可被引用，无论，构造对象下，还是原型对象  
//那么问题来了？？？？   既然绝对会用为何不直接声明在sim.APP下面？？  
    this.container = container;  
    this.renderer = renderer;  
    this.scene = scene;  
    this.camera = camera;  
    this.projector = projector;  
    this.root = root;               //可被渲染的object3D存在这里  
      
    // Set up event handlers  
    this.initMouse();  
    this.initKeyboard();  
    this.addDomHandlers();  
}  
  
  
  
  
  
//Core run loop循环  
Sim.App.prototype.run = function()//this会被绑定  
{  
    this.update();                    //Sim.App下的原型对象，可以调用其他原型对象？ 调用自己所在的构造函数下的原型函数，??????  
                                      //问题解决：这个run（）会在后期调用，自己会被绑定到this上面，所以可以   
  
    this.renderer.render( this.scene, this.camera );//这里的两个this和上面一个道理  
    var that = this;  
    requestAnimationFrame(function() { that.run(); });    
}  
  
// Update method - called once per tick  
Sim.App.prototype.update = function()//这个在App原型对象里面的run（）里面调用  ？？问题：同一个对象下面的原型函数可以互相调用？？  
                                     //问题解决：同一个对象下面的原型函数的确可以互相调用，就像这里的run()可以调用update()  
{  
    var i, len;  
    len = this.objects.length;  
    for (i = 0; i < len; i++)  
    {  
        this.objects[i].update();//这里的update什么意思？从哪里来的？  
                                 //问题解决：这里的update是object自带的；object来自继承Sim.Object  
    }  
}  
  
  
// Add/remove objects  
//以下是this.object的入口  
  
Sim.App.prototype.addObject = function(obj)//因为前面的scene camera renderer全部在原型对象的init里面初始化了，  
                                           //所以这里主要是mesh   
{  
    this.objects.push(obj);  
  
    // If this is a renderable object, add it to the root scene  
    //若从obj里面检查到object3D说明其可被渲染  
    if (obj.object3D)  
    {  
        this.root.add(obj.object3D);  
    }  
}  
  
Sim.App.prototype.removeObject = function(obj)  
{  
    var index = this.objects.indexOf(obj);  
    if (index != -1)  
    {  
        this.objects.splice(index, 1);//删除object[]中的元素  
        // If this is a renderable object, remove it from the root scene  
        if (obj.object3D)  
        {  
            this.root.remove(obj.object3D);//这里的this属于原型调用原型  
        }  
    }  
}  
  
// Event handling  
Sim.App.prototype.initMouse = function()  
{  
    var dom = this.renderer.domElement;  
      
    var that = this;  
    //添加鼠标移动的事件  
    dom.addEventListener( 'mousemove',   
            function(e) { that.onDocumentMouseMove(e); }, false );  
  
    //添加鼠标按下的事件  
    dom.addEventListener( 'mousedown',   
            function(e) { that.onDocumentMouseDown(e); }, false );  
    //添加鼠标弹起的事件  
    dom.addEventListener( 'mouseup',   
            function(e) { that.onDocumentMouseUp(e); }, false );  
  
    //中键滑轮？？  
    $(dom).mousewheel(  
            function(e, delta) {  
                that.onDocumentMouseScroll(e, delta);  
            }  
        );  
      
    this.overObject = null;  
    this.clickedObject = null;  
}  
  
Sim.App.prototype.initKeyboard = function()  
{  
    var dom = this.renderer.domElement;  
      
    var that = this;  
    dom.addEventListener( 'keydown',   
            function(e) { that.onKeyDown(e); }, false );  
    dom.addEventListener( 'keyup',   
            function(e) { that.onKeyUp(e); }, false );  
    dom.addEventListener( 'keypress',   
            function(e) { that.onKeyPress(e); }, false );  
  
    // so it can take focus  
    dom.setAttribute("tabindex", 1);  
    dom.style.outline='none';  
}  
  
Sim.App.prototype.addDomHandlers = function()  
{  
    var that = this;  
    window.addEventListener( 'resize', function(event) { that.onWindowResize(event); }, false );  
}  
  
Sim.App.prototype.onDocumentMouseMove = function(event)  
{  
    event.preventDefault();  
      
    if (this.clickedObject && this.clickedObject.handleMouseMove)  
    {  
        var hitpoint = null, hitnormal = null;  
        var intersected = this.objectFromMouse(event.pageX, event.pageY);  
        if (intersected.object == this.clickedObject)  
        {  
            hitpoint = intersected.point;  
            hitnormal = intersected.normal;  
        }  
        this.clickedObject.handleMouseMove(event.pageX, event.pageY, hitpoint, hitnormal);  
    }  
    else  
    {  
        var handled = false;  
          
        var oldObj = this.overObject;  
        var intersected = this.objectFromMouse(event.pageX, event.pageY);  
        this.overObject = intersected.object;  
      
        if (this.overObject != oldObj)  
        {  
            if (oldObj)  
            {  
                this.container.style.cursor = 'auto';  
                  
                if (oldObj.handleMouseOut)  
                {  
                    oldObj.handleMouseOut(event.pageX, event.pageY);  
                }  
            }  
      
            if (this.overObject)  
            {  
                if (this.overObject.overCursor)  
                {  
                    this.container.style.cursor = this.overObject.overCursor;  
                }  
                  
                if (this.overObject.handleMouseOver)  
                {  
                    this.overObject.handleMouseOver(event.pageX, event.pageY);  
                }  
            }  
              
            handled = true;  
        }  
      
        if (!handled && this.handleMouseMove)  
        {  
            this.handleMouseMove(event.pageX, event.pageY);  
        }  
    }  
}  
  
Sim.App.prototype.onDocumentMouseDown = function(event)  
{  
    event.preventDefault();//阻止元素默认动作，好用来执行以下代码  
          
    var handled = false;  
  
    var intersected = this.objectFromMouse(event.pageX, event.pageY);  
    if (intersected.object)  
    {  
        if (intersected.object.handleMouseDown)  
        {  
            intersected.object.handleMouseDown(event.pageX, event.pageY, intersected.point, intersected.normal);  
            this.clickedObject = intersected.object;  
            handled = true;  
        }  
    }  
      
    if (!handled && this.handleMouseDown)  
    {  
        this.handleMouseDown(event.pageX, event.pageY);  
    }  
}  
  
Sim.App.prototype.onDocumentMouseUp = function(event)  
{  
    event.preventDefault();  
      
    var handled = false;  
      
    var intersected = this.objectFromMouse(event.pageX, event.pageY);  
    if (intersected.object)  
    {  
        if (intersected.object.handleMouseUp)  
        {  
            intersected.object.handleMouseUp(event.pageX, event.pageY, intersected.point, intersected.normal);  
            handled = true;  
        }  
    }  
      
    if (!handled && this.handleMouseUp)  
    {  
        this.handleMouseUp(event.pageX, event.pageY);  
    }  
      
    this.clickedObject = null;  
}  
  
Sim.App.prototype.onDocumentMouseScroll = function(event, delta)  
{  
    event.preventDefault();  
  
    if (this.handleMouseScroll)  
    {  
        this.handleMouseScroll(delta);  
    }  
}  
  
Sim.App.prototype.objectFromMouse = function(pagex, pagey)  
{  
    // Translate page coords to element coords  
    var offset = $(this.renderer.domElement).offset();    
    var eltx = pagex - offset.left;  
    var elty = pagey - offset.top;  
      
    // Translate client coords into viewport x,y  
    var vpx = ( eltx / this.container.offsetWidth ) * 2 - 1;  
    var vpy = - ( elty / this.container.offsetHeight ) * 2 + 1;  
      
    var vector = new THREE.Vector3( vpx, vpy, 0.5 );  
  
    this.projector.unprojectVector( vector, this.camera );  
      
    var ray = new THREE.Ray( this.camera.position, vector.subSelf( this.camera.position ).normalize() );  
  
    var intersects = ray.intersectScene( this.scene );  
      
    if ( intersects.length > 0 ) {         
          
        var i = 0;  
        while(!intersects[i].object.visible)  
        {  
            i++;  
        }  
          
        var intersected = intersects[i];  
        var mat = new THREE.Matrix4().getInverse(intersected.object.matrixWorld);  
        var point = mat.multiplyVector3(intersected.point);  
          
        return (this.findObjectFromIntersected(intersected.object, intersected.point, intersected.face.normal));                                                   
    }  
    else  
    {  
        return { object : null, point : null, normal : null };  
    }  
}  
  
Sim.App.prototype.findObjectFromIntersected = function(object, point, normal)  
{  
    if (object.data)  
    {  
        return { object: object.data, point: point, normal: normal };  
    }  
    else if (object.parent)  
    {  
        return this.findObjectFromIntersected(object.parent, point, normal);  
    }  
    else  
    {  
        return { object : null, point : null, normal : null };  
    }  
}  
  
  
Sim.App.prototype.onKeyDown = function(event)  
{  
    // N.B.: Chrome doesn't deliver keyPress if we don't bubble... keep an eye on this  
    event.preventDefault();  
  
    if (this.handleKeyDown)  
    {  
        this.handleKeyDown(event.keyCode, event.charCode);  
    }  
}  
  
Sim.App.prototype.onKeyUp = function(event)  
{  
    // N.B.: Chrome doesn't deliver keyPress if we don't bubble... keep an eye on this  
    event.preventDefault();  
  
    if (this.handleKeyUp)  
    {  
        this.handleKeyUp(event.keyCode, event.charCode);  
    }  
}  
              
Sim.App.prototype.onKeyPress = function(event)  
{  
    // N.B.: Chrome doesn't deliver keyPress if we don't bubble... keep an eye on this  
    event.preventDefault();  
  
    if (this.handleKeyPress)  
    {  
        this.handleKeyPress(event.keyCode, event.charCode);  
    }  
}  
  
Sim.App.prototype.onWindowResize = function(event) {  
  
    this.renderer.setSize(this.container.offsetWidth, this.container.offsetHeight);  
  
    this.camera.aspect = this.container.offsetWidth / this.container.offsetHeight;  
    this.camera.updateProjectionMatrix();  
  
}  
  
Sim.App.prototype.focus = function()  
{  
    if (this.renderer && this.renderer.domElement)  
    {  
        this.renderer.domElement.focus();  
    }  
}  
  
  
  
  
  
  
  
  
// Sim.Object - base class for all objects in our simulation  
Sim.Object = function()  
{  
    Sim.Publisher.call(this);  
      
    this.object3D = null;      //入口在Sim.Object.prototype.setObject3D  
    this.children = [];        //入口在Sim.Object.prototype.addChild  
}  
  
Sim.Object.prototype = new Sim.Publisher;  
  
Sim.Object.prototype.init = function()  
{  
}  
  
Sim.Object.prototype.update = function()                            //那么现在大问题来了有两个Sim.Object.prototype.update  
{  
    this.updateChildren();  
}  
  
// setPosition - move the object to a new position  
Sim.Object.prototype.setPosition = function(x, y, z)  
{  
    if (this.object3D)  
    {  
        this.object3D.position.set(x, y, z);  
    }  
}  
  
//setScale - scale the object  
Sim.Object.prototype.setScale = function(x, y, z)  
{  
    if (this.object3D)  
    {  
        this.object3D.scale.set(x, y, z);  
    }  
}  
  
//setScale - scale the object  
Sim.Object.prototype.setVisible = function(visible)  
{  
    function setVisible(obj, visible)  
    {  
        obj.visible = visible;  
        var i, len = obj.children.length;  
        for (i = 0; i < len; i++)  
        {  
            setVisible(obj.children[i], visible);  
        }  
    }  
      
    if (this.object3D)  
    {  
        setVisible(this.object3D, visible);  
    }  
}  
  
// updateChildren - update all child objects  
Sim.Object.prototype.update = function()                             //那么现在大问题来了有两个Sim.Object.prototype.update  
{  
    var i, len;  
    len = this.children.length;  
    for (i = 0; i < len; i++)  
    {  
        this.children[i].update();  
    }  
}  
  
Sim.Object.prototype.setObject3D = function(object3D)  
{  
    object3D.data = this;      //原型对象中的this，到底指的是那个this？到底是Sim.Object还是Sim.Object.prototype.setObject3D  
                               //解决：经试验是Sim.Object的this  
  
    this.object3D = object3D;  
}  
  
//Add/remove children  
Sim.Object.prototype.addChild = function(child)  
{  
    this.children.push(child);  
      
    // If this is a renderable object, add its object3D as a child of mine  
    if (child.object3D)  
    {  
        this.object3D.add(child.object3D);  
    }  
}  
  
Sim.Object.prototype.removeChild = function(child)  
{  
    var index = this.children.indexOf(child);  
    if (index != -1)  
    {  
        this.children.splice(index, 1);  
        // If this is a renderable object, remove its object3D as a child of mine  
        if (child.object3D)  
        {  
            this.object3D.remove(child.object3D);  
        }  
    }  
}  
  
// Some utility methods  
Sim.Object.prototype.getScene = function()  
{  
    var scene = null;  
    if (this.object3D)  
    {  
        var obj = this.object3D;  
        while (obj.parent)  
        {  
            obj = obj.parent;//若obj一直存在parent则，一直把obj.parent赋值给obj，直到obj不存在parent，  
                             //说明找到了最上面的那个obj  
                             //那么问题来了：所有的object的父节点就是scene？？？  
        }  
          
        scene = obj;  
    }  
      
    return scene;  
}  
  
Sim.Object.prototype.getApp = function()  
{  
    var scene = this.getScene();  
    return scene ? scene.data : null;  
}  
  
// Some constants  
  
/* key codes 
37: left 
38: up 
39: right 
40: down 
*/  
  
/** 
 * [KeyCodes description]以下为sim.js的键盘keycode编号，直接定义为全局变量方便使用，也便于理解 
 * @type {Object} 
 */  
Sim.KeyCodes = {};  
Sim.KeyCodes.KEY_LEFT  = 37;//左键  
Sim.KeyCodes.KEY_UP  = 38;//上键  
Sim.KeyCodes.KEY_RIGHT  = 39;//右键  
Sim.KeyCodes.KEY_DOWN  = 40;//下键  
hello world

