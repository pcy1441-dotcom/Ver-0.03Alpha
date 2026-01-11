<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">

<title>머더 역할+능력 추첨기</title>

<style>
body {
  font-family: Arial, sans-serif;
  background: #f4f4f4;
  margin: 0;
  padding: 20px;
  text-align: center;
  color: black;
}

h1 {
  font-size: clamp(24px, 6vw, 36px);
  margin-bottom: 20px;
}

input[type=number] {
  padding: 12px;
  font-size: 18px;
  width: 80px;
  text-align: center;
}

button {
  padding: 16px 28px;
  margin: 12px;
  font-size: clamp(16px,4.5vw,20px);
  cursor: pointer;
  border-radius: 10px;
  border: none;
}

#menu, #draw-area {
  margin-top: 40px;
}

#result {
  display: none;
  background: white;
  padding: 35px 20px;
  border-radius: 16px;
  margin-top: 40px;
}

#skill, #role {
  font-size: clamp(28px, 8vw, 46px);
  font-weight: bold;
  margin-bottom: 15px;
}

#desc {
  font-size: clamp(16px, 4.5vw, 20px);
  line-height: 1.4;
  margin-bottom: 30px;
}

.role-murder { color: #FF1801; }
.role-sheriff { color: #3712FF; }
.role-citizen { color: #18FF39; }

/* 능력 색상 */
.skill-yellow { color: #FFDC00; }
.skill-pink { color: #FB66FF; }

/* 총잡이 대각선 색상 */
.skill-gunslinger {
  background: linear-gradient(45deg, #FF1801 50%, #3712FF 50%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  display: inline-block;
}
</style>
</head>
<body>

<h1>머더 역할+능력 추첨기</h1>

<div id="menu">
  <label>인원 수 입력: <input type="number" id="playerCount" min="4" max="20" value="6"></label>
  <button onclick="startGame()">시작</button>
</div>

<div id="draw-area" style="display:none;">
  <div id="playerNum"></div>
  <button onclick="draw()">뽑기</button>
</div>

<div id="result">
  <div id="role"></div>
  <div id="skill"></div>
  <div id="desc"></div>
  <button onclick="nextRound()">다음</button>
</div>

<script>
const skills = {
  머더: [
    { name: "방탄", desc: "한 번 공격을 버틸 수 있습니다.(2목숨)" },
    { name: "가방", desc: "죽인 대상이 살아있는 것처럼 행동하다가 마지막에 사망합니다.(1명 한정)" },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "총잡이", desc: "보안관과 같은 방식으로 처치합니다.(어깨 터치)", special: "gunslinger" },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "무능력", desc: "능력이 없습니다." }
  ],
  보안관: [
    { name: "강인한", desc: "목숨이 2개가 됩니다." },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "마피아", desc: "머더와 모든 시민을 제거하면 즉시 승리합니다.", special: "yellow" },
    { name: "무능력", desc: "능력이 없습니다." }
  ],
  의사: [
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "자가치유", desc: "자신을 치료할 수 있습니다.(1회 한정)" },
    { name: "전문 의사", desc: "최대 2명까지 치료할 수 있습니다." },
    { name: "구급상자", desc: "생존자 1명에게 미리 지급해 스스로 치료할 수 있게 합니다.(머더 포함)" },
    { name: "무능력", desc: "능력이 없습니다." }
  ],
  시민: [
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "스파이", desc: "시민/보안관을 죽일수록 최대 2목숨까지 증가합니다. 머더를 죽이면 자신도 사망하며 시민 승리.", special: "yellow" },
    { name: "퍼블win", desc: "가장 먼저 머더에게 죽으면 즉시 시민이 승리합니다.", special: "pink" },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "무능력", desc: "능력이 없습니다." },
    { name: "패링", desc: "한 번의 공격을 막아냅니다." }
  ]
};

let totalPlayers = 0;
let currentPlayer = 0;
let roleOrder = [];
let usedSpecialCitizen = [];

function startGame() {
  totalPlayers = parseInt(document.getElementById("playerCount").value);
  if(isNaN(totalPlayers) || totalPlayers<4) { alert("최소 4명 이상 입력하세요."); return; }

  // 역할 배열 생성
  roleOrder = ["머더", "보안관", "의사"];
  for(let i=4; i<=totalPlayers; i++) roleOrder.push("시민");

  // 섞기
  roleOrder = roleOrder.sort(() => Math.random()-0.5);
  currentPlayer = 1;
  usedSpecialCitizen = [];

  document.getElementById("menu").style.display = "none";
  document.getElementById("draw-area").style.display = "block";
  document.getElementById("playerNum").textContent = `플레이어 ${currentPlayer}번째`;
}

function draw() {
  const role = roleOrder[currentPlayer-1];
  const abilityList = [...skills[role]];

  let chosen;

  if(role === "시민") {
    // 시민 특수능력 최대 2명
    let specials = abilityList.filter(a => a.special && !usedSpecialCitizen.includes(a.name));
    if(specials.length > 0 && usedSpecialCitizen.length < 2) {
      chosen = specials[Math.floor(Math.random() * specials.length)];
      usedSpecialCitizen.push(chosen.name);
    } else {
      // 나머지는 무능력
      chosen = abilityList.find(a => !a.special || a.name === "무능력");
    }
  } else {
    chosen = abilityList[Math.floor(Math.random() * abilityList.length)];
  }

  const roleDiv = document.getElementById("role");
  const skillDiv = document.getElementById("skill");
  const descDiv = document.getElementById("desc");

  // 역할 색상
  roleDiv.textContent = role;
  roleDiv.className = role === "머더" ? "role-murder" :
                      role === "보안관" ? "role-sheriff" : "role-citizen";

  // 능력 색상
  if(chosen.special === "yellow") skillDiv.className = "skill-yellow";
  else if(chosen.special === "pink") skillDiv.className = "skill-pink";
  else if(chosen.special === "gunslinger") skillDiv.className = "skill-gunslinger";
  else skillDiv.className = "";

  skillDiv.textContent = chosen.name;
  descDiv.textContent = chosen.desc;

  document.getElementById("draw-area").style.display = "none";
  document.getElementById("result").style.display = "block";
}

function nextRound() {
  currentPlayer++;
  if(currentPlayer > totalPlayers){
    alert("모든 플레이어 완료! 게임 초기화됩니다.");
    document.getElementById("result").style.display = "none";
    document.getElementById("draw-area").style.display = "none";
    document.getElementById("menu").style.display = "block";
    return;
  }
  document.getElementById("result").style.display = "none";
  document.getElementById("draw-area").style.display = "block";
  document.getElementById("playerNum").textContent = `플레이어 ${currentPlayer}번째`;
}
</script>

</body>
</html>
