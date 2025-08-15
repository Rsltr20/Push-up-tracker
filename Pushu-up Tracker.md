<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Push-Up Tracker</title>
<style>
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(to bottom right, #4facfe, #00f2fe);
    color: #fff;
    text-align: center;
    margin: 0;
    padding: 0;
}
h1 { margin-top: 20px; }
#goalInputContainer, #trackerContainer {
    margin: 20px auto;
    max-width: 320px;
    background: rgba(255,255,255,0.1);
    padding: 20px;
    border-radius: 15px;
}
input[type=number] {
    width: 80%;
    padding: 10px;
    border-radius: 10px;
    border: none;
    font-size: 1.2em;
    text-align: center;
}
button {
    margin-top: 10px;
    padding: 10px 20px;
    border: none;
    border-radius: 10px;
    background: #ffcc00;
    color: #333;
    font-size: 1em;
    cursor: pointer;
    transition: all 0.2s;
}
button:hover { transform: scale(1.05); }
.progress-bar {
    height: 20px;
    width: 100%;
    background: rgba(255,255,255,0.3);
    border-radius: 10px;
    margin: 15px 0;
    overflow: hidden;
}
.progress { height: 100%; width: 0%; background: #ffcc00; border-radius: 10px; transition: width 0.3s; }
#streak, #score, #monthlyProgress, #yearlyProgress { font-size: 1.2em; margin-top: 10px; }
canvas { position: fixed; top:0; left:0; width:100%; height:100%; pointer-events:none; }
</style>
</head>
<body>

<h1>Push-Up Tracker 💪</h1>

<div id="goalInputContainer">
    <p>Set your daily push-up goal:</p>
    <input type="number" id="dailyGoal" min="1" placeholder="e.g. 250">
    <br>
    <button id="setGoal">Set Goal</button>
</div>

<div id="trackerContainer" style="display:none;">
    <h2 id="todayCount">0 / 0</h2>
    <div class="progress-bar"><div class="progress" id="progressBar"></div></div>
    <button id="addPushup">Add Push-Up</button>
    <p id="streak">Streak: 0 days</p>
    <p id="score">Score: 0</p>
    <button id="shareStreak" style="background:#00ccff;">Share Your Streak</button>
    <p id="monthlyProgress">Monthly: 0 / 0</p>
    <div class="progress-bar"><div class="progress" id="monthlyBar"></div></div>
    <p id="yearlyProgress">Yearly: 0 / 0</p>
    <div class="progress-bar"><div class="progress" id="yearlyBar"></div></div>
    <button id="resetDay" style="background:#ff6666;">Reset Today</button>
    <button id="changeGoal" style="background:#66ff66;">Change Goal</button>
</div>

<canvas id="confetti"></canvas>

<script>
const today = new Date();
const todayStr = today.toISOString().split('T')[0];
const yearStart = new Date(today.getFullYear(),0,1);
const monthStart = new Date(today.getFullYear(),today.getMonth(),1);
const storageKey = 'pushupTracker';

// --- Confetti Function ---
function launchConfetti(duration=3000){
    const canvas = document.getElementById('confetti');
    const ctx = canvas.getContext('2d');
    canvas.width = window.innerWidth; canvas.height = window.innerHeight;
    const particles = [];
    for(let i=0;i<100;i++){
        particles.push({
            x: Math.random()*canvas.width,
            y: Math.random()*canvas.height,
            dx: (Math.random()-0.5)*5,
            dy: Math.random()*5+2,
            r: Math.random()*5+2,
            color:`hsl(${Math.random()*360},100%,50%)`
        });
    }
    const start = Date.now();
    function animate(){
        ctx.clearRect(0,0,canvas.width,canvas.height);
        particles.forEach(p=>{
            p.x += p.dx; p.y += p.dy; p.dy += 0.1;
            if(p.y>canvas.height){ p.y=0; p.x=Math.random()*canvas.width; p.dy=Math.random()*5+2; }
            ctx.fillStyle=p.color; ctx.beginPath(); ctx.arc(p.x,p.y,p.r,0,2*Math.PI); ctx.fill();
        });
        if(Date.now()-start<duration) requestAnimationFrame(animate);
        else ctx.clearRect(0,0,canvas.width,canvas.height);
    }
    animate();
}

// --- Local Storage Functions ---
function loadData(){ return JSON.parse(localStorage.getItem(storageKey)) || null; }
function saveData(data){ localStorage.setItem(storageKey, JSON.stringify(data)); }

// --- Initialize Tracker ---
function init(){
    const data = loadData();
    if(!data){
        document.getElementById('goalInputContainer').style.display='block';
        document.getElementById('trackerContainer').style.display='none';
    } else {
        if(!data.progress) data.progress={};
        if(!data.progress[todayStr]) data.progress[todayStr]=0;
        if(!data.streak) data.streak=0;
        if(!data.score) data.score=0;
        if(!data.monthlyMilestones) data.monthlyMilestones={};
        if(!data.yearlyMilestone) data.yearlyMilestone=false;
        if(!data.streakMilestones) data.streakMilestones={};
        showTracker(data);
    }
}

// --- Update Display ---
function showTracker(data){
    document.getElementById('goalInputContainer').style.display='none';
    document.getElementById('trackerContainer').style.display='block';
    
    const todayCount = data.progress[todayStr] || 0;
    document.getElementById('todayCount').innerText=`${todayCount} / ${data.dailyGoal}`;
    document.getElementById('streak').innerText=`Streak: ${data.streak} days`;
    document.getElementById('score').innerText=`Score: ${data.score}`;
    
    // Monthly Progress
    const monthSum = Object.entries(data.progress).filter(([date])=>{
        const d=new Date(date); return d>=monthStart;
    }).reduce((a,[_,v])=>a+v,0);
    const monthGoal = data.dailyGoal * (new Date(today.getFullYear(),today.getMonth()+1,0).getDate());
    document.getElementById('monthlyProgress').innerText=`Monthly: ${monthSum} / ${monthGoal}`;
    document.getElementById('monthlyBar').style.width=Math.min((monthSum/monthGoal)*100,100)+'%';
    
    // Yearly Progress
    const yearSum = Object.entries(data.progress).filter(([date])=>{
        const d=new Date(date); return d>=yearStart;
    }).reduce((a,[_,v])=>a+v,0);
    const yearGoal = data.dailyGoal * 365;
    document.getElementById('yearlyProgress').innerText=`Yearly: ${yearSum} / ${yearGoal}`;
    document.getElementById('yearlyBar').style.width=Math.min((yearSum/yearGoal)*100,100)+'%';
}

// --- Event Listeners ---
document.getElementById('setGoal').addEventListener('click',()=>{
    const goal=parseInt(document.getElementById('dailyGoal').value);
    if(goal>0){
        const data = { dailyGoal:goal, progress:{[todayStr]:0}, streak:0, score:0, lastDate:todayStr, monthlyMilestones:{}, yearlyMilestone:false, streakMilestones:{} };
        saveData(data); showTracker(data);
    } else { alert('Enter a valid number'); }
});

document.getElementById('addPushup').addEventListener('click',()=>{
    const data=loadData();
    if(!data.progress[todayStr]) data.progress[todayStr]=0;
    data.progress[todayStr]++;
    
    // Update streak
    const yesterday = new Date(Date.now()-86400000).toISOString().split('T')[0];
    if(data.lastDate!==todayStr){
        if(data.progress[yesterday]>=data.dailyGoal) data.streak++; else data.streak=1;
        data.lastDate=todayStr;
    }
    
    // Update score
    const overGoal = data.progress[todayStr]>data.dailyGoal?data.progress[todayStr]-data.dailyGoal:0;
    data.score = Object.values(data.progress).reduce((a,b)=>a+b,0) + overGoal*2;

    // --- Streak Bonuses ---
    const streakMilestones = [7,30,100];
    streakMilestones.forEach(ms=>{
        if(data.streak>=ms && !data.streakMilestones[ms]){
            data.streakMilestones[ms]=true;
            data.score += ms*2; // bonus points
            launchConfetti(ms===100?5000:3000);
            alert(`🔥 Streak Milestone! ${ms} days streak bonus unlocked!`);
        }
    });

    // --- Monthly milestone ---
    const monthSum = Object.entries(data.progress).filter(([date])=>{
        const d=new Date(date); return d>=monthStart;
    }).reduce((a,[_,v])=>a+v,0);
    const monthGoal = data.dailyGoal * (new Date(today.getFullYear(),today.getMonth()+1,0).getDate());
    if(monthSum >= monthGoal && !data.monthlyMilestones[today.getMonth()]){
        data.monthlyMilestones[today.getMonth()]=true; launchConfetti(); alert("🎉 Monthly milestone unlocked!");
    }

    // --- Yearly milestone ---
    const yearSum = Object.values(data.progress).reduce((a,b)=>a+b,0);
    const yearGoal = data.dailyGoal * 365;
    if(yearSum >= yearGoal && !data.yearlyMilestone){
        data.yearlyMilestone=true; launchConfetti(5000); alert("🏆 Yearly milestone unlocked!");
    }

    saveData(data); showTracker(data);
});

// --- Reset Day & Change Goal ---
document.getElementById('resetDay').addEventListener('click',()=>{
    const data=loadData(); data.progress[todayStr]=0; saveData(data); showTracker(data);
});
document.getElementById('changeGoal').addEventListener('click',()=>{
    document.getElementById('goalInputContainer').style.display='block';
    document.getElementById('trackerContainer').style.display='none';
});

// --- Share Streak ---
document.getElementById('shareStreak').addEventListener('click', ()=>{
    const data = loadData();
    const streak = data.streak || 0;
    const score = data.score || 0;
    const dailyGoal = data.dailyGoal;
    const message = `💪 I’m on a ${streak}-day push-up streak today with a score of ${score}! Can you beat me? Daily goal: ${dailyGoal} push-ups!`;
    navigator.clipboard.writeText(message).then(()=>{
        alert('✅ Streak copied! Send it to a friend to challenge them!');
        launchConfetti(2000);
    }).catch(err=>{
        alert('Oops, failed to copy. Just screenshot your streak!');
    });
});

init();
</script>
</body>
</html>