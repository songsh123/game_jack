<template>
  <div id="app">
    <div class="ambient ambient-one"></div>
    <div class="ambient ambient-two"></div>

    <main class="shell">
      <header class="brand">
        <div class="brand-mark"><span>J</span><span>J</span></div>
        <div>
          <p class="eyebrow">CARD GAME SCORE KEEPER</p>
          <h1>JackJack</h1>
        </div>
        <button v-if="gameStarted" class="icon-button" aria-label="설정" @click="showSettings = true">⚙</button>
      </header>

      <section v-if="!gameStarted" class="setup-card card">
        <div class="section-heading">
          <span class="step">01</span>
          <div><h2>게임을 시작해볼까요?</h2><p>함께할 플레이어와 기본 판돈을 설정하세요.</p></div>
        </div>

        <label class="field-label">플레이어</label>
        <div class="player-input" v-for="(name, index) in setupNames" :key="index">
          <span class="player-number">{{ index + 1 }}</span>
          <input v-model.trim="setupNames[index]" :placeholder="`플레이어 ${index + 1} 이름`" @keyup.enter="addSetupPlayer">
          <button v-if="setupNames.length > 2" aria-label="플레이어 삭제" @click="removeSetupPlayer(index)">×</button>
        </div>
        <button class="add-player" @click="addSetupPlayer"><span>＋</span> 플레이어 추가</button>

        <label class="field-label stake-label">기본 판돈</label>
        <div class="money-input">
          <input v-model.number="setupStake" inputmode="numeric" type="number" min="100" step="100">
          <span>원</span>
        </div>
        <div class="quick-stakes">
          <button v-for="amount in quickStakes" :key="amount" :class="{ active: setupStake === amount }" @click="setupStake = amount">{{ formatCompact(amount) }}</button>
        </div>

        <p v-if="setupError" class="error-message">{{ setupError }}</p>
        <button class="primary-button" @click="startGame">게임 시작 <span>→</span></button>
      </section>

      <template v-else>
        <section class="score-section">
          <div class="score-title-row">
            <div><p class="eyebrow">LIVE SCORE</p><h2>현재 스코어</h2></div>
            <div class="round-pill"><span></span>{{ history.length + 1 }}번째 판</div>
          </div>
          <div class="score-grid">
            <article v-for="(player, index) in sortedPlayers" :key="player.id" class="score-card" :class="scoreClass(player.score)">
              <div class="rank">{{ index + 1 }}</div>
              <div class="avatar">{{ player.name.charAt(0) }}</div>
              <p>{{ player.name }}</p>
              <strong>{{ signedMoney(player.score) }}</strong>
            </article>
          </div>
          <div class="zero-check"><span>✓</span> 전체 합계 {{ formatMoney(totalScore) }} · 계산 정상</div>
        </section>

        <section class="round-card card">
          <div class="section-heading compact">
            <span class="step">02</span>
            <div><h2>이번 판 결과</h2><p>승자와 패자를 눌러 선택하세요.</p></div>
          </div>

          <div class="role-legend"><span class="winner-dot"></span>승자 <span class="loser-dot"></span>패자 <span class="neutral-dot"></span>참여 없음</div>
          <div class="role-list">
            <button v-for="player in players" :key="player.id" class="role-row" :class="roles[player.id]" @click="cycleRole(player.id)">
              <span class="avatar small">{{ player.name.charAt(0) }}</span>
              <strong>{{ player.name }}</strong>
              <span class="role-label">{{ roleLabel(roles[player.id]) }}</span>
              <span class="role-icon">{{ roleIcon(roles[player.id]) }}</span>
            </button>
          </div>

          <div class="stake-row">
            <div><span>이번 판돈</span><small>이 판에만 변경할 수 있어요</small></div>
            <div class="inline-money"><input v-model.number="roundStake" type="number" min="100" step="100" inputmode="numeric"><span>원</span></div>
          </div>

          <div v-if="previewTransfers.length" class="preview-box">
            <p>예상 이동</p>
            <div v-for="(transfer, index) in previewTransfers" :key="index" class="transfer-line">
              <span>{{ transfer.from }}</span><b>→</b><span>{{ transfer.to }}</span><strong>{{ formatMoney(transfer.amount) }}</strong>
            </div>
          </div>
          <p v-else class="selection-guide">승자와 패자를 각각 한 명 이상 선택해 주세요.</p>

          <button class="primary-button" :disabled="!canApplyRound" @click="applyRound">결과 적용 <span>✓</span></button>
        </section>

        <section class="history-section">
          <div class="history-header"><div><p class="eyebrow">GAME LOG</p><h2>판 기록</h2></div><button v-if="history.length" @click="undoLast">↶ 마지막 판 취소</button></div>
          <div v-if="!history.length" class="empty-history">아직 기록된 판이 없어요.<br>첫 번째 결과를 입력해 보세요!</div>
          <article v-for="(round, index) in reversedHistory" :key="round.id" class="history-item">
            <div class="history-round">{{ history.length - index }}</div>
            <div><strong>{{ roundSummary(round) }}</strong><p>{{ formatMoney(round.stake) }} · {{ round.transfers.length }}건 이동</p></div>
            <span>{{ round.time }}</span>
          </article>
        </section>

        <button class="settle-button" :disabled="!history.length" @click="showSettlement = true"><span>₩</span><div><strong>게임 끝내고 정산하기</strong><small>최소 송금 횟수로 정리해드려요</small></div><b>→</b></button>
      </template>
    </main>

    <div v-if="showSettings" class="modal-backdrop" @click.self="showSettings = false">
      <section class="modal card">
        <button class="modal-close" @click="showSettings = false">×</button>
        <p class="eyebrow">GAME SETTINGS</p><h2>게임 설정</h2>
        <label class="field-label">기본 판돈</label>
        <div class="money-input"><input v-model.number="defaultStake" type="number" min="100" step="100"><span>원</span></div>
        <button class="secondary-button" @click="resetRoles">현재 선택 초기화</button>
        <button class="danger-button" @click="newGame">새 게임 시작</button>
      </section>
    </div>

    <div v-if="showSettlement" class="modal-backdrop" @click.self="showSettlement = false">
      <section class="modal settlement-modal card">
        <button class="modal-close" @click="showSettlement = false">×</button>
        <p class="eyebrow">FINAL SETTLEMENT</p><h2>최종 정산</h2><p class="modal-copy">아래대로 송금하면 깔끔하게 끝나요.</p>
        <div v-if="settlements.length" class="settlement-list">
          <div v-for="(item, index) in settlements" :key="index">
            <span class="avatar small">{{ item.from.charAt(0) }}</span><strong>{{ item.from }}</strong><b>→</b><strong>{{ item.to }}</strong><em>{{ formatMoney(item.amount) }}</em>
          </div>
        </div>
        <div v-else class="empty-history">모두 0원이에요. 정산할 금액이 없습니다!</div>
        <button class="primary-button" @click="showSettlement = false">확인</button>
      </section>
    </div>
  </div>
</template>

<script>
const STORAGE_KEY = 'jackjack-game-v1'

export default {
  name: 'App',
  data () {
    return {
      gameStarted: false,
      setupNames: ['짱구', '철수', '맹구'],
      setupStake: 2000,
      defaultStake: 2000,
      roundStake: 2000,
      quickStakes: [1000, 2000, 5000, 10000],
      players: [],
      roles: {},
      history: [],
      setupError: '',
      showSettings: false,
      showSettlement: false
    }
  },
  computed: {
    sortedPlayers () { return [...this.players].sort((a, b) => b.score - a.score) },
    totalScore () { return this.players.reduce((sum, player) => sum + player.score, 0) },
    winners () { return this.players.filter(player => this.roles[player.id] === 'winner') },
    losers () { return this.players.filter(player => this.roles[player.id] === 'loser') },
    canApplyRound () { return this.winners.length > 0 && this.losers.length > 0 && this.roundStake > 0 },
    previewTransfers () {
      const transfers = []
      this.losers.forEach(loser => this.winners.forEach(winner => transfers.push({ from: loser.name, to: winner.name, fromId: loser.id, toId: winner.id, amount: Number(this.roundStake) })))
      return transfers
    },
    reversedHistory () { return [...this.history].reverse() },
    settlements () {
      const debtors = this.players.filter(p => p.score < 0).map(p => ({ name: p.name, amount: -p.score }))
      const creditors = this.players.filter(p => p.score > 0).map(p => ({ name: p.name, amount: p.score }))
      const result = []; let d = 0; let c = 0
      while (d < debtors.length && c < creditors.length) {
        const amount = Math.min(debtors[d].amount, creditors[c].amount)
        result.push({ from: debtors[d].name, to: creditors[c].name, amount })
        debtors[d].amount -= amount; creditors[c].amount -= amount
        if (debtors[d].amount === 0) d++
        if (creditors[c].amount === 0) c++
      }
      return result
    }
  },
  watch: {
    defaultStake (value) { if (value > 0) { this.roundStake = value; this.saveGame() } },
    showSettings (open) { if (!open) this.saveGame() }
  },
  mounted () { this.loadGame() },
  methods: {
    addSetupPlayer () { if (this.setupNames.length < 12) this.setupNames.push('') },
    removeSetupPlayer (index) { this.setupNames.splice(index, 1) },
    startGame () {
      const names = this.setupNames.map(name => name.trim()).filter(Boolean)
      if (names.length < 2) { this.setupError = '플레이어 이름을 두 명 이상 입력해 주세요.'; return }
      if (new Set(names).size !== names.length) { this.setupError = '플레이어 이름은 서로 다르게 입력해 주세요.'; return }
      if (!this.setupStake || this.setupStake <= 0) { this.setupError = '판돈을 올바르게 입력해 주세요.'; return }
      this.players = names.map((name, index) => ({ id: Date.now() + index, name, score: 0 }))
      this.defaultStake = Number(this.setupStake); this.roundStake = Number(this.setupStake)
      this.gameStarted = true; this.setupError = ''; this.resetRoles(); this.saveGame()
    },
    cycleRole (id) {
      const next = { neutral: 'winner', winner: 'loser', loser: 'neutral' }
      this.$set(this.roles, id, next[this.roles[id] || 'neutral'])
    },
    resetRoles () { const roles = {}; this.players.forEach(player => { roles[player.id] = 'neutral' }); this.roles = roles },
    applyRound () {
      if (!this.canApplyRound) return
      const transfers = this.previewTransfers.map(item => ({ ...item }))
      transfers.forEach(item => {
        this.players.find(p => p.id === item.fromId).score -= item.amount
        this.players.find(p => p.id === item.toId).score += item.amount
      })
      this.history.push({ id: Date.now(), stake: Number(this.roundStake), transfers, winners: this.winners.map(p => p.name), losers: this.losers.map(p => p.name), time: new Date().toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit' }) })
      this.roundStake = this.defaultStake; this.resetRoles(); this.saveGame()
    },
    undoLast () {
      const round = this.history.pop(); if (!round) return
      round.transfers.forEach(item => {
        this.players.find(p => p.id === item.fromId).score += item.amount
        this.players.find(p => p.id === item.toId).score -= item.amount
      })
      this.saveGame()
    },
    newGame () {
      if (!window.confirm('현재 게임 기록을 지우고 새로 시작할까요?')) return
      localStorage.removeItem(STORAGE_KEY); this.gameStarted = false; this.players = []; this.history = []; this.showSettings = false; this.setupError = ''
    },
    saveGame () {
      if (!this.gameStarted) return
      localStorage.setItem(STORAGE_KEY, JSON.stringify({ gameStarted: true, players: this.players, history: this.history, defaultStake: this.defaultStake, roundStake: this.roundStake }))
    },
    loadGame () {
      try {
        const saved = JSON.parse(localStorage.getItem(STORAGE_KEY))
        if (!saved || !saved.gameStarted) return
        this.gameStarted = true; this.players = saved.players || []; this.history = saved.history || []; this.defaultStake = saved.defaultStake || 2000; this.roundStake = saved.roundStake || this.defaultStake; this.resetRoles()
      } catch (error) { localStorage.removeItem(STORAGE_KEY) }
    },
    roleLabel (role) { return { winner: '승자', loser: '패자', neutral: '참여 없음' }[role || 'neutral'] },
    roleIcon (role) { return { winner: 'W', loser: 'L', neutral: '−' }[role || 'neutral'] },
    roundSummary (round) { return `${round.winners.join('·')} 승 / ${round.losers.join('·')} 패` },
    formatMoney (amount) { return `${Number(amount).toLocaleString('ko-KR')}원` },
    signedMoney (amount) { return amount > 0 ? `+${this.formatMoney(amount)}` : this.formatMoney(amount) },
    formatCompact (amount) { return amount >= 10000 ? `${amount / 10000}만원` : `${amount / 1000}천원` },
    scoreClass (score) { return score > 0 ? 'positive' : score < 0 ? 'negative' : 'zero' }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@500;600;700;800&family=Noto+Sans+KR:wght@400;500;600;700;800&display=swap');
:root { --bg:#111218; --card:#1c1e27; --card2:#232630; --line:#30333f; --text:#f5f3ed; --muted:#9698a3; --lime:#c8f561; --green:#65d9a2; --red:#ff776f; --gold:#ffc96b; }
* { box-sizing:border-box; }
body { margin:0; background:var(--bg); color:var(--text); font-family:'Noto Sans KR',sans-serif; -webkit-font-smoothing:antialiased; }
button,input { font:inherit; }
button { cursor:pointer; }
#app { min-height:100vh; overflow:hidden; position:relative; padding:36px 16px 80px; }
.ambient { position:fixed; border-radius:50%; filter:blur(110px); pointer-events:none; opacity:.13; }
.ambient-one { width:350px;height:350px;background:var(--lime);top:-190px;left:-100px; }
.ambient-two { width:400px;height:400px;background:#795cff;right:-260px;top:45%; }
.shell { max-width:720px; margin:0 auto; position:relative; z-index:1; }
.brand { display:flex; align-items:center; gap:13px; margin-bottom:30px; }
.brand-mark { width:47px;height:47px;border:1px solid #3c3f4b;border-radius:14px;background:#20222b;display:flex;align-items:center;justify-content:center;transform:rotate(-4deg);box-shadow:0 10px 30px #0005; }
.brand-mark span { font:800 19px 'DM Sans';margin:-2px;color:var(--lime); }
.brand-mark span:last-child { color:#fff; transform:translateY(6px); }
.brand h1 { font:800 25px 'DM Sans';margin:2px 0 0;letter-spacing:-.8px; }
.eyebrow { margin:0;color:var(--lime);font:700 10px 'DM Sans';letter-spacing:1.8px; }
.icon-button { margin-left:auto;width:43px;height:43px;border-radius:13px;border:1px solid var(--line);background:var(--card);color:#bbb;font-size:19px; }
.card { background:linear-gradient(150deg,#1e2029,#191b23);border:1px solid var(--line);border-radius:24px;padding:27px;box-shadow:0 28px 80px #0004; }
.section-heading { display:flex;gap:15px;align-items:flex-start;margin-bottom:27px; }
.section-heading.compact { margin-bottom:20px; }
.step { flex:0 0 38px;height:38px;border-radius:11px;background:var(--lime);color:#151714;display:grid;place-items:center;font:800 12px 'DM Sans'; }
h2 { margin:0;font-size:21px;letter-spacing:-.7px; }
.section-heading p,.modal-copy { margin:6px 0 0;color:var(--muted);font-size:13px; }
.field-label { display:block;color:#c8c9cf;font-size:12px;font-weight:700;margin:19px 0 9px; }
.player-input { display:flex;align-items:center;gap:10px;margin-bottom:9px; }
.player-number { width:30px;text-align:center;color:#696c78;font:700 12px 'DM Sans'; }
input { width:100%;background:#14161d;border:1px solid #343743;border-radius:13px;color:#f5f5f5;outline:none;padding:14px 15px;transition:.2s; }
input:focus { border-color:var(--lime);box-shadow:0 0 0 3px #c8f56116; }
.player-input button { border:0;background:transparent;color:#767985;font-size:22px;padding:5px; }
.add-player { margin:4px 0 0 40px;border:0;background:transparent;color:var(--lime);font-size:12px;font-weight:700; }
.stake-label { margin-top:27px; }
.money-input { display:flex;align-items:center;background:#14161d;border:1px solid #343743;border-radius:13px;padding-right:15px; }
.money-input input { border:0;background:transparent;font:700 19px 'DM Sans'; }
.money-input span { color:var(--muted);font-size:13px; }
.quick-stakes { display:grid;grid-template-columns:repeat(4,1fr);gap:7px;margin-top:9px; }
.quick-stakes button { padding:9px 4px;border:1px solid var(--line);border-radius:10px;background:#242630;color:#9b9da7;font-size:11px;font-weight:700; }
.quick-stakes button.active { border-color:var(--lime);color:var(--lime);background:#c8f5610c; }
.primary-button { width:100%;border:0;border-radius:14px;background:var(--lime);color:#151713;padding:16px;margin-top:24px;font-weight:800;box-shadow:0 10px 30px #c8f56118; }
.primary-button span { float:right;font-size:18px;line-height:18px; }
.primary-button:disabled { opacity:.3;cursor:not-allowed;box-shadow:none; }
.error-message { color:var(--red);font-size:12px;margin:15px 0 -8px; }
.score-section { margin-bottom:24px; }
.score-title-row,.history-header { display:flex;justify-content:space-between;align-items:flex-end;margin-bottom:15px;padding:0 3px; }
.score-title-row h2,.history-header h2 { margin-top:4px; }
.round-pill { border:1px solid var(--line);border-radius:20px;padding:8px 11px;color:#bfc0c6;font-size:11px;font-weight:600;background:#1b1d25; }
.round-pill span { display:inline-block;width:6px;height:6px;border-radius:50%;background:var(--lime);box-shadow:0 0 8px var(--lime);margin-right:6px; }
.score-grid { display:grid;grid-template-columns:repeat(3,1fr);gap:9px; }
.score-card { min-width:0;position:relative;text-align:center;background:linear-gradient(145deg,#242731,#1b1d25);border:1px solid var(--line);border-radius:18px;padding:17px 7px 16px;overflow:hidden; }
.score-card.positive { border-color:#4d6f5e; }.score-card.negative { border-color:#684743; }
.rank { position:absolute;top:8px;left:10px;color:#686a75;font:800 10px 'DM Sans'; }
.avatar { margin:0 auto 7px;width:42px;height:42px;display:grid;place-items:center;border-radius:14px;background:linear-gradient(145deg,#3c4050,#282b36);color:#ddd;font-weight:800; }
.positive .avatar { background:#345c4b;color:#8cf0bc; }.negative .avatar { background:#5b3939;color:#ff9d97; }
.score-card p { font-size:12px;margin:0 0 8px;color:#d6d6da;font-weight:700;white-space:nowrap;overflow:hidden;text-overflow:ellipsis; }
.score-card strong { font:700 15px 'DM Sans';white-space:nowrap; }.positive strong { color:var(--green); }.negative strong { color:var(--red); }
.zero-check { color:#777a84;text-align:center;font-size:10px;margin-top:10px; }.zero-check span { color:var(--green); }
.round-card { margin-bottom:30px; }
.role-legend { font-size:10px;color:#777a83;text-align:right;margin:-5px 2px 13px; }
.role-legend span { display:inline-block;width:6px;height:6px;border-radius:50%;margin-left:8px;margin-right:3px; }.winner-dot{background:var(--green)}.loser-dot{background:var(--red)}.neutral-dot{background:#5a5d68}
.role-list { display:grid;gap:8px; }
.role-row { width:100%;display:flex;align-items:center;gap:11px;padding:10px 12px;border:1px solid var(--line);background:#171920;color:#ddd;border-radius:14px;text-align:left; }
.avatar.small { margin:0;width:34px;height:34px;border-radius:11px;font-size:12px; }
.role-row strong { font-size:13px; }.role-label { margin-left:auto;font-size:11px;color:#7f818b; }.role-icon { width:28px;height:28px;border-radius:9px;display:grid;place-items:center;background:#272a34;color:#777;font:800 11px 'DM Sans'; }
.role-row.winner { border-color:#4b745f;background:#1b2925; }.role-row.winner .role-label,.role-row.winner .role-icon { color:var(--green); }.role-row.winner .role-icon { background:#284c3e; }
.role-row.loser { border-color:#704b49;background:#2c2021; }.role-row.loser .role-label,.role-row.loser .role-icon { color:var(--red); }.role-row.loser .role-icon { background:#533130; }
.stake-row { display:flex;align-items:center;justify-content:space-between;border-top:1px solid var(--line);margin-top:19px;padding-top:18px;gap:16px; }
.stake-row>div>span { display:block;font-size:12px;font-weight:700; }.stake-row small { display:block;color:#747681;font-size:9px;margin-top:4px; }
.inline-money { display:flex;align-items:center;max-width:155px; }.inline-money input { text-align:right;padding:11px 7px;font-weight:700; }.inline-money span { margin-left:7px;color:#999;font-size:12px; }
.preview-box { margin-top:16px;padding:14px;background:#14161c;border-radius:13px; }.preview-box>p { margin:0 0 9px;color:#888b96;font-size:10px;font-weight:700; }
.transfer-line { display:grid;grid-template-columns:1fr 18px 1fr auto;gap:6px;align-items:center;font-size:11px;margin-top:6px; }.transfer-line b { color:#666; }.transfer-line strong { color:var(--gold);font:700 11px 'DM Sans'; }
.selection-guide { margin:15px 0 -8px;text-align:center;color:#70727d;font-size:11px; }
.history-section { margin:0 2px 28px; }.history-header button { border:0;background:transparent;color:#9597a1;font-size:10px; }
.empty-history { border:1px dashed #343741;border-radius:17px;padding:28px;text-align:center;color:#676a74;font-size:12px;line-height:1.8; }
.history-item { display:flex;align-items:center;gap:12px;padding:13px 3px;border-bottom:1px solid #272a33; }.history-round { width:31px;height:31px;border-radius:10px;background:#242731;display:grid;place-items:center;color:var(--lime);font:800 11px 'DM Sans'; }.history-item strong{font-size:11px}.history-item p{margin:3px 0 0;color:#70737d;font-size:9px}.history-item>span{margin-left:auto;color:#696c76;font-size:9px}
.settle-button { width:100%;display:flex;align-items:center;gap:13px;padding:17px;border:1px solid #414536;border-radius:18px;background:linear-gradient(135deg,#282d21,#1e211d);color:#eee;text-align:left; }.settle-button>span { width:39px;height:39px;border-radius:12px;background:var(--lime);color:#171914;display:grid;place-items:center;font-weight:800; }.settle-button div { display:flex;flex-direction:column;gap:4px; }.settle-button strong{font-size:13px}.settle-button small{color:#81847b;font-size:9px}.settle-button>b{margin-left:auto;color:var(--lime)}.settle-button:disabled{opacity:.35;cursor:not-allowed}
.modal-backdrop { position:fixed;z-index:10;inset:0;background:#08090cdd;display:grid;place-items:center;padding:18px;backdrop-filter:blur(8px); }.modal { position:relative;width:min(440px,100%); }.modal h2{margin:5px 0 25px}.modal-close{position:absolute;right:18px;top:15px;border:0;background:transparent;color:#888b94;font-size:25px}.secondary-button,.danger-button{width:100%;padding:13px;margin-top:12px;border-radius:13px;background:#242731;border:1px solid var(--line);color:#ddd;font-weight:700;font-size:12px}.danger-button{color:var(--red);border-color:#593b3d;background:#2d2022}.settlement-list{display:grid;gap:8px;margin:20px 0}.settlement-list>div{display:grid;grid-template-columns:34px 1fr 20px 1fr auto;align-items:center;gap:8px;padding:11px;background:#15171e;border-radius:13px;font-size:11px}.settlement-list b{color:#656873}.settlement-list em{font-style:normal;color:var(--gold);font:700 11px 'DM Sans'}
@media (min-width:620px){#app{padding-top:54px}.setup-card{padding:36px}.score-grid{gap:13px}.score-card{padding:22px 10px}.score-card strong{font-size:18px}.round-card{padding:32px}.role-list{grid-template-columns:repeat(2,1fr)}}
@media (max-width:370px){.card{padding:21px 17px}.score-grid{gap:6px}.score-card strong{font-size:12px}.quick-stakes{grid-template-columns:repeat(2,1fr)}}
</style>
