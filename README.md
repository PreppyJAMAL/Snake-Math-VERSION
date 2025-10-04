<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Flappy Math</title>
<style>
    body {
        margin: 0;
        overflow: hidden;
        background: #87ceeb;
        font-family: "Comic Sans MS", cursive, sans-serif;
    }
    canvas { display: block; }
    #restart, #wrongRestart {
        font-size: 28px;
        padding: 15px 25px;
        background: red;
        color: white;
        border: none;
        border-radius: 10px;
        cursor: pointer;
        font-family: "Comic Sans MS", cursive, sans-serif;
    }
    #restart { display: none; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); }
    #answerBox {
        display: none;
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background-color: #001f4d;
        color: white;
        padding: 30px;
        border-radius: 15px;
        text-align: center;
        font-family: "Comic Sans MS", cursive, sans-serif;
    }
    #answerBox button {
        font-size: 24px;
        padding: 12px 20px;
        margin: 10px;
        cursor: pointer;
        border: none;
        border-radius: 8px;
        background-color: #004080;
        color: white;
    }
    #scoreDisplay {
        position: absolute;
        top: 20px;
        left: 50%;
        transform: translateX(-50%);
        font-size: 64px;
        font-weight: bold;
        color: black;
        text-shadow: 2px 2px white;
        font-family: "Comic Sans MS", cursive, sans-serif;
    }
    #directionsBox {
        position: absolute;
        top: 20px;
        left: 20px;
        background-color: white;
        padding: 15px;
        border-radius: 10px;
        font-size: 18px;
        max-width: 300px;
        font-family: "Comic Sans MS", cursive, sans-serif;
        color: black;
    }
</style>
</head>
<body>

<canvas id="game"></canvas>
<button id="restart">Restart</button>
<div id="scoreDisplay">0</div>
<div id="directionsBox">
    Tap or press SPACE to flap! Navigate the pipes and solve math questions on each pipe. Wrong answers show a restart option.
</div>

<div id="answerBox">
    <div id="questionText" style="margin-bottom:20px;font-size:24px;"></div>
    <div id="choices"></div>
    <button id="wrongRestart" style="display:none;background:red;">Restart</button>
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// Fullscreen canvas
function resizeCanvas(){
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas();

const bird = { x: 100, y: canvas.height/2, width: 40, height: 30, velocity: 0, gravity: 0.1, jump: -6 };
let pipes = [], score = 0, gameOver = false, frame = 0, paused = false, currentPipe = null;

const scoreDisplay = document.getElementById("scoreDisplay");
const answerBox = document.getElementById("answerBox");
const questionText = document.getElementById("questionText");
const choicesDiv = document.getElementById("choices");
const restartBtn = document.getElementById("restart");
const wrongRestartBtn = document.getElementById("wrongRestart");

function flap(){ if(!gameOver && !paused) bird.velocity = bird.jump; }

// Controls: Spacebar & touch
document.addEventListener("keydown", e => { if(e.code === "Space") flap(); });
canvas.addEventListener("touchstart", e => { e.preventDefault(); flap(); }, {passive:false});

function generateQuestion(){
    const a=Math.floor(Math.random()*10+1), b=Math.floor(Math.random()*10+1);
    const operators=["+", "-", "X", "÷"];
    const op = operators[Math.floor(Math.random()*4)];
    let answer;
    if(op==="+") answer=a+b;
    if(op==="-") answer=a-b;
    if(op==="X") answer=a*b;
    if(op==="÷") answer=Math.floor(a/b);

    const choices=[answer];
    while(choices.length<4){
        let wrong=answer+Math.floor(Math.random()*10-5);
        if(!choices.includes(wrong)) choices.push(wrong);
    }
    choices.sort(()=>Math.random()-0.5);
    return {text:`${a} ${op} ${b} = ?`, answer, choices};
}

function createPipe(){
    const pipeHeight=Math.floor(Math.random()*(canvas.height/2-150)+100);
    const question=generateQuestion();
    return {x:canvas.width,width:80,gap:400,top:pipeHeight,bottom:canvas.height-pipeHeight-400,question:question.text,answer:question.answer,choices:question.choices,passed:false,topHitbox:{},bottomHitbox:{}};
}

restartBtn.addEventListener("click", resetGame);
wrongRestartBtn.addEventListener("click", resetGame);

function resetGame(){
    bird.y=canvas.height/2; bird.velocity=0;
    pipes=[]; score=0; frame=0; gameOver=false; paused=false; currentPipe=null;
    restartBtn.style.display="none"; wrongRestartBtn.style.display="none"; answerBox.style.display="none";
    scoreDisplay.innerText="0"; animate();
}

function showChoices(pipe){
    paused=true; currentPipe=pipe;
    questionText.innerText=pipe.question;
    choicesDiv.innerHTML="";
    wrongRestartBtn.style.display="none";
    pipe.choices.forEach(choice=>{
        const btn=document.createElement("button");
        btn.innerText=choice;
        btn.addEventListener("click", ()=>{
            if(choice===pipe.answer){
                paused=false; answerBox.style.display="none"; pipe.passed=true;
                score++; scoreDisplay.innerText=score;
            } else {
                choicesDiv.innerHTML=`<p>Wrong answer Lil Bru, the correct answer is ${pipe.answer}</p>`;
                wrongRestartBtn.style.display="block";
            }
        });
        choicesDiv.appendChild(btn);
    });
    answerBox.style.display="block";
}

function drawBird(){
    ctx.save();
    ctx.translate(bird.x+bird.width/2,bird.y+bird.height/2);
    ctx.rotate(Math.min(Math.max(bird.velocity/8,-0.5),0.5));
    ctx.fillStyle="orange";
    ctx.beginPath(); ctx.ellipse(0,0,bird.width/2,bird.height/2,0,0,Math.PI*2); ctx.fill();
    ctx.fillStyle="white"; ctx.beginPath(); ctx.arc(5,-5,5,0,Math.PI*2); ctx.fill();
    ctx.restore();
}

function rectIntersect(a,b){return a.x<b.x+b.width && a.x+a.width>b.x && a.y<b.y+b.height && a.y+a.height>b.y;}

function animate(){
    if(gameOver) return;
    if(paused){ requestAnimationFrame(animate); return; }

    ctx.clearRect(0,0,canvas.width,canvas.height);
    bird.velocity+=bird.gravity; bird.y+=bird.velocity;
    drawBird();

    if(frame%300===0) pipes.push(createPipe());

    for(let i=pipes.length-1;i>=0;i--){
        const p=pipes[i];
        p.x-=1;

        ctx.fillStyle="green";
        ctx.fillRect(p.x,0,p.width,p.top);
        ctx.fillRect(p.x,canvas.height-p.bottom,p.width,p.bottom);

        p.topHitbox={x:p.x,y:0,width:p.width,height:p.top};
        p.bottomHitbox={x:p.x,y:canvas.height-p.bottom,width:p.width,height:p.bottom};

        ctx.fillStyle="white"; ctx.font="20px Comic Sans MS";
        ctx.fillText(p.question,p.x+5,p.top+25);

        if(rectIntersect(bird,p.topHitbox)||rectIntersect(bird,p.bottomHitbox)){
            gameOver=true; restartBtn.style.display="block";
        }

        if(!p.passed && bird.x+bird.width>p.x && bird.x<p.x+p.width) showChoices(p);
        if(p.x+p.width<0) pipes.splice(i,1);
    }

    if(bird.y+bird.height>canvas.height || bird.y<0){ gameOver=true; restartBtn.style.display="block"; }

    frame++; requestAnimationFrame(animate);
}

animate();
</script>
</body>
</html>
