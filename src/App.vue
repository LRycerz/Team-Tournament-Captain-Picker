<template>
  <div class="inputs">
    <div class="input-row">
      <div class="input-block">
        <label for="our">Our Team</label>
        <textarea id="our" v-model="ourText" rows="5" placeholder="One team per line"></textarea>
      </div>
      <div class="input-block">
        <label for="enemy">Enemy Team</label>
        <textarea id="enemy" v-model="enemyText" rows="5" placeholder="One team per line"></textarea>
      </div>
      <div class="input-block">
        <label for="maps">Maps</label>
        <textarea id="maps" v-model="mapsText" rows="5" placeholder="One map per line"></textarea>
      </div>
    </div>
    <button @click="generateMatrix">Send data</button>
  </div>

  <div v-if="matrix.length" class="matrix">
    <table border="1">
      <thead>
        <tr>
          <th></th>
          <th v-for="(enemy, j) in enemyTeams" :key="`e${j}`">{{ enemy }}</th>
          <th v-for="(map, k) in maps" :key="`m${k}`">{{ map }}</th>
          <th v-for="(mission, l) in missions" :key="`ms${l}`">{{ mission }}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(our, i) in ourTeams" :key="i">
          <th>{{ our }}</th>
          <td v-for="(cell, j) in matrix[i]" :key="j">
            <input v-model="matrix[i][j]" />
          </td>
        </tr>
      </tbody>
      <tfoot>
        <tr>
          <th>Total</th>
          <td v-for="(sum, idx) in columnSums" :key="idx">{{ sum }}</td>
        </tr>
      </tfoot>
    </table>
  </div>

  <!-- selection workflow section -->
  <section v-if="ourTeams.length && enemyTeams.length" class="selection">
    <h2>Pairing Selection</h2>

    <!-- Step 1: determine defender/attacker and ban missions -->
    <div v-if="pairingStep === 0">
      <h3>Step 1 – Defender &amp; Attacker</h3>
      <p>Choose which side starts as Defender (A). The other side will be Attacker (B).</p>
      <label>
        Defender side:
        <select v-model="defenderSide">
          <option value="our">Our Team</option>
          <option value="enemy">Enemy Team</option>
        </select>
      </label>
      <p>You may ban missions from the pool:</p>
      <div class="ban-section">
        <label>
          ({{ teamForLetter('A') }}) Defender bans two missions:
          <select v-model="bannedA" multiple size="9">
            <option v-for="m in missions" :key="m" :value="m">{{ m }}</option>
          </select>
        </label>
        <label>
          ({{ teamForLetter('B') }}) Attacker bans one mission:
          <select v-model="bannedB" size="9">
            <option v-for="m in missions" :key="m" :value="m">{{ m }}</option>
          </select>
        </label>
      </div>
      <p v-if="ourTeams.length">Suggestion to ban: {{ suggestedMissionBan() }} (least preferred by our team)</p>
      <button @click="applyBans(); resetPools(); nextPairing()">Continue to Pairing 1</button>
    </div>

    <!-- subsequent pairing steps -->
    <div v-else>
      
        <h3>Pairing {{ pairingStep }}</h3>
        <p>Defender: {{ pairingStep % 2 === 1 ? 'A' : 'B' }} – Attacker: {{ pairingStep % 2 === 1 ? 'B' : 'A' }}</p>
        <!-- choose kill teams and map/mission according to the rules -->
        <div class="pairing-actions">
          <div class="pairing-step"> 
            <p >1. ({{ teamForLetter('A') }}) Defender chooses 1 kill team from their pool:</p>
            <select size="5" v-model="pairingData['defenderChoice' + pairingStep]">
              <option v-for="team in poolA" :key="team" :value="team">{{ team }}</option>
            </select>
          </div>
          <div class="pairing-step">
            <p>2. ({{ teamForLetter('B') }}) Attacker chooses 2 kill teams from their pool:</p>
                        <p v-if="sideForRole('enemy')">
              Suggestion: {{ suggestAttackerSelection(pairingData['defenderChoice' + pairingStep]) }}
            </p>
            <div>
              <select multiple size="5" v-model="pairingData['attackerOptions' + pairingStep]">
                <option v-for="team in poolB" :key="team" :value="team">{{ team }}</option>
              </select>
            </div>
          </div>
            <div class="pairing-step">


          <p>3. ({{ teamForLetter('A') }}) Defender then picks one of the above for the matchup:</p>
                      <p v-if="sideForRole('our')">
              Suggestion: {{ suggestDefenderChoice(pairingData['defenderChoice' + pairingStep], pairingData['attackerOptions' + pairingStep] || []) }}
            </p>
          <select v-model="pairingData['attackerSelected' + pairingStep]">
            <option v-for="team in pairingData['attackerOptions' + pairingStep] || []" :key="team" :value="team">{{ team }}</option>
          </select>
            </div>
          <div class="pairing-step">
          <p>4. ({{ teamForLetter('B') }}) Attacker chooses a map from available maps:</p>
          <p v-if="sideForRole('enemy') && pairingData['attackerSelected' + pairingStep]">
            Suggested map: {{ recommendMap(pairingData['attackerSelected' + pairingStep], mapsPool) }}
          </p>
          <select v-model="pairingData['map' + pairingStep]">
            <option v-for="map in mapsPool" :key="map" :value="map">{{ map }}</option>
          </select>
          
          </div>
          <div class="pairing-step">
          <p>5. ({{ teamForLetter('A') }}) Defender chooses a mission from pool:</p>
          <p v-if="sideForRole('our')">Suggested mission: {{ recommendMission(pairingData['defenderChoice' + pairingStep], missionsPool) }}</p>
          <select v-model="pairingData['mission' + pairingStep]">
            <option v-for="m in missionsPool" :key="m" :value="m">{{ m }}</option>
          </select>
          
          </div>
          <div class="pairing-step" v-if="pairingStep === 3">
            <p >({{ teamForLetter('A') }}) Defender now bans another mission for the remaining process:</p>
            <select v-model="pairingData.extraBan3">
              <option value="" disabled>Select mission to ban</option>
              <option v-for="m in missionsPool" :key="m" :value="m">{{ m }}</option>
            </select>
          </div>
        </div>

      <button @click="prevPairing">Previous</button>
      <button @click="nextPairing">Next</button>
    </div>
    <div v-if="pairingStep > 5">
      <h3>Selection Complete</h3>
      <p>The pairing process is finished. Follow the initiative rules:</p>
      <ul>
        <li>(B) chooses one pairing in which they have won the first initiative roll.</li>
        <li>(A) chooses three pairings in which they have won the first initiative roll.</li>
        <li>The final pairing is then determined as usual.</li>
      </ul>
    </div>
  </section>
</template>

<script>
import { ref, watch, onMounted, reactive, computed } from 'vue';

export default {
  name: 'App',
  setup() {
    const ourText = ref('');
    const enemyText = ref('');
    const mapsText = ref('');

    const ourTeams = ref([]);
    const enemyTeams = ref([]);
    const maps = ref([]);
    const matrix = ref([]);

    // static missions list to append after maps in header
    const missions = [
      'Secure',
      'Loot',
      'Transmission',
      'Orb',
      'Stake Claim',
      'Energy Cells',
      'Download',
      'Data',
      'Reboot',
    ];

    function generateMatrix() {
      ourTeams.value = ourText.value
        .split(/\r?\n/)
        .map((s) => s.trim())
        .filter((s) => s);
      enemyTeams.value = enemyText.value
        .split(/\r?\n/)
        .map((s) => s.trim())
        .filter((s) => s);

      maps.value = mapsText.value
        .split(/\r?\n/)
        .map((s) => s.trim())
        .filter((s) => s);

      // build matrix of empty strings with extra columns for maps + missions
      const cols =
        enemyTeams.value.length + maps.value.length + missions.length;
      matrix.value = ourTeams.value.map(() => Array.from({ length: cols }, () => ''));
      // initialize selection pools now that we know the teams/maps
      resetPools();
    }

    // computed helper for column totals
    const columnSums = computed(() => {
      if (!matrix.value.length) return [];
      const cols = matrix.value[0].length;
      const sums = Array.from({ length: cols }, () => 0);
      matrix.value.forEach((row) => {
        row.forEach((cell, j) => {
          const val = parseFloat(cell);
          if (!isNaN(val)) sums[j] += val;
        });
      });
      return sums;
    });

    // persist data to localStorage whenever relevant state changes
    const saveState = () => {
      const payload = {
        ourText: ourText.value,
        enemyText: enemyText.value,
        mapsText: mapsText.value,
        matrix: matrix.value,
      };
      localStorage.setItem('pickerState', JSON.stringify(payload));
    };

    watch([ourText, enemyText, mapsText, matrix], saveState, { deep: true });

    // restore on mount
    onMounted(() => {
      try {
        const saved = JSON.parse(localStorage.getItem('pickerState') || '{}');
        if (saved.ourText) ourText.value = saved.ourText;
        if (saved.enemyText) enemyText.value = saved.enemyText;
        if (saved.mapsText) mapsText.value = saved.mapsText;
        // regenerate teams and matrix from text if present
        if (saved.ourText || saved.enemyText) {
          generateMatrix();
          if (saved.matrix) {
            matrix.value = saved.matrix;
          }
        }
      } catch (e) {
        console.warn('failed to restore state', e);
      }
    });

    // watch the matrix for changes and print priority arrays for each row
    watch(
      matrix,
      (newVal) => {
        // for each our team row compute sorted lists
        newVal.forEach((row, i) => {
          const enemyCount = enemyTeams.value.length;
          const mapCount = maps.value.length;

          // helper to build sorted name list given start index and length
          const buildList = (start, length, names) => {
            const items = [];
            for (let j = 0; j < length; j++) {
              const val = parseFloat(row[start + j]);
              items.push({ name: names[j], priority: isNaN(val) ? Infinity : val });
            }
            items.sort((a, b) => a.priority - b.priority);
            return items.map((it) => it.name);
          };

          const enemyOrder = buildList(0, enemyCount, enemyTeams.value);
          const mapOrder = buildList(enemyCount, mapCount, maps.value);
          const missionOrder = buildList(enemyCount + mapCount, missions.length, missions);

          // store suggestions for this our team
          suggestions.value[ourTeams.value[i]] = {
            enemyOrder,
            mapOrder,
            missionOrder,
          };
        });
      },
      { deep: true }
    );

    // suggestion storage keyed by our team name
    const suggestions = ref({});

    // helpers to recommend best option given candidates
    function recommendEnemy(team, candidates) {
      const order = suggestions.value[team]?.enemyOrder || [];
      let best = candidates[0];
      let bestIdx = order.length;
      candidates.forEach((c) => {
        const idx = order.indexOf(c);
        if (idx !== -1 && idx < bestIdx) {
          bestIdx = idx;
          best = c;
        }
      });
      return best;
    }
    function recommendMap(team, candidates) {
      const order = suggestions.value[team]?.mapOrder || [];
      let best = candidates[0];
      let bestIdx = order.length;
      candidates.forEach((c) => {
        const idx = order.indexOf(c);
        if (idx !== -1 && idx < bestIdx) {
          bestIdx = idx;
          best = c;
        }
      });
      return best;
    }
    function recommendMission(team, candidates) {
      const order = suggestions.value[team]?.missionOrder || [];
      let best = candidates[0];
      let bestIdx = order.length;
      candidates.forEach((c) => {
        const idx = order.indexOf(c);
        if (idx !== -1 && idx < bestIdx) {
          bestIdx = idx;
          best = c;
        }
      });
      return best;
    }

    // suggestion helpers for pairing step
    function suggestDefenderChoice(defenderTeam, options) {
      // when our team is defender, options are enemy teams attacker selected
      if (!options || !options.length) return '';
      // use the specific defender team if provided, otherwise fall back to first team
      const team = defenderTeam || ourTeams.value[0];
      return recommendEnemy(team, options);
    }

    function suggestAttackerSelection(enemyDefenderTeam) {
      // when enemy is defender, pick the two of our teams that most prefer
      // to face that enemy defender (i.e. have that enemy highest in their enemyOrder)
      if (!enemyDefenderTeam) return '';
      const available = poolB.value.slice();
      // score each available our-team by where they rank the enemyDefenderTeam
      const scored = available.map((ourTeam) => {
        const order = suggestions.value[ourTeam]?.enemyOrder || [];
        const idx = order.indexOf(enemyDefenderTeam);
        return { ourTeam, idx: idx === -1 ? Infinity : idx };
      });
      // sort ascending (lower index == higher preference)
      scored.sort((a, b) => a.idx - b.idx);
      // pick top two that actually ranked the enemy (idx !== Infinity)
      const picks = scored.filter((s) => s.idx !== Infinity).slice(0, 2).map((s) => s.ourTeam);
      // if fewer than two found, fill from remaining available teams
      if (picks.length < 2) {
        const remaining = available.filter((t) => !picks.includes(t)).slice(0, 2 - picks.length);
        picks.push(...remaining);
      }
      return picks.join(', ');
    }

    function suggestedMissionBan() {
      if (!ourTeams.value.length) return '';
      const scores = {};
      missions.forEach((m) => (scores[m] = 0));
      ourTeams.value.forEach((team) => {
        const order = suggestions.value[team]?.missionOrder || [];
        missions.forEach((m) => {
          const pos = order.indexOf(m);
          scores[m] += pos === -1 ? missions.length : pos;
        });
      });
      const sorted = [...missions].sort((a, b) => scores[b] - scores[a]);
      if (defenderSide.value === 'our') {
        return sorted.slice(0, 2).join(', ');
      }
      return sorted[0];
    }

    // determine which side (our/enemy) holds a given role for current step
    // function sideForRole(role) {
    //   if (pairingStep.value === 0) return null;
    //   const defIsA = pairingStep.value % 2 === 1;
    //   const defSide = defIsA ? 'A' : 'B';
    //   const attSide = defIsA ? 'B' : 'A';
    //   const sideMap = { A: defenderSide.value === 'our' ? 'our' : 'enemy', B: defenderSide.value === 'our' ? 'enemy' : 'our' };
    //   return role === 'defender' ? sideMap[defSide] : sideMap[attSide];
    // }

    function sideForRole(role) {
      if (defenderSide.value === role) {
        return true;
      }
      else {
        return false;
      }
    }

    // function isOurTurn(role) {
    //   return sideForRole(role) === 'our';
    // }

    function teamForLetter(letter) {
      if (defenderSide.value === 'our') {
        return letter === 'A' ? 'Our Team' : 'Enemy Team';
      }
      return letter === 'A' ? 'Enemy Team' : 'Our Team';
    }

            // --- selection section state ---
            const defenderSide = ref('our'); // 'our' or 'enemy' - which list is A/Defender initially
            const poolA = ref([]);
            const poolB = ref([]);
            const missionsPool = ref([...missions]);
            const mapsPool = ref([]);

            const bannedA = ref([]);
            const bannedB = ref([]);

            const pairingStep = ref(0);
            const pairingData = reactive({}); // store choices per step

            // helpers to preserve pools across defenderSide flips
            const preservePools = ref(false);
            const savedPools = { a: [], b: [] };
            // helper to reset pools when teams or defenderSide change
            function resetPools() {
              // console.log('resetPools called');
              // console.log('current pools before reset', {preservePools, poolA: poolA.value, poolB: poolB.value, mapsPool: mapsPool.value, missionsPool: missionsPool.value });
              if (preservePools.value) {
                poolA.value = [...savedPools.a];
                poolB.value = [...savedPools.b];
                // console.log('pools preserved from saved');
                preservePools.value = false;
                return;
              }
              if (defenderSide.value === 'our') {
                poolA.value = [...ourTeams.value];
                poolB.value = [...enemyTeams.value];
              } else {
                poolA.value = [...enemyTeams.value];
                poolB.value = [...ourTeams.value];
              }
              mapsPool.value = [...maps.value];
              missionsPool.value = [...missions.filter((m) => !bannedA.value.includes(m) && !bannedB.value.includes(m))];
              // console.log('pools reset', { poolA: poolA.value, poolB: poolB.value, mapsPool: mapsPool.value, missionsPool: missionsPool.value });
            }

            watch([ourTeams, enemyTeams, defenderSide], resetPools);

            function applyBans() {
              // console.log('applyBans called');
              missionsPool.value = missions.filter((m) => !bannedA.value.includes(m) && !bannedB.value.includes(m));
            }

            function nextPairing() {
              const step = pairingStep.value;

              // when moving from step 0 to 1 we already handled bans earlier in the button
              if (step > 0) {
                // TUTAJ 2
                // const def = step % 2 === 1 ? 'A' : 'B';
                // const att = def === 'A' ? 'B' : 'A';
                const def = 'A';
                const att = 'B';
                console.log('pools:',{poolA: poolA, poolB: poolB});
                const defenderPool = def === 'A' ? poolA : poolB;
                const attackerPool = att === 'A' ? poolA : poolB;
                console.log('def/att pools:', {defenderPool: defenderPool.value, attackerPool: attackerPool.value});
                const defenderChoice = pairingData['defenderChoice' + step];
                const attackerOptions = pairingData['attackerOptions' + step] || [];
                const attackerSelected = pairingData['attackerSelected' + step];
                const mapChoice = pairingData['map' + step];
                const missionChoice = pairingData['mission' + step];
                console.log('nextPairing before', {pairingData, step, defenderChoice, attackerOptions, attackerSelected, mapChoice, missionChoice });
                // remove defender's pick from their pool
                if (defenderChoice) {
                  
                  const idx = defenderPool.value.indexOf(defenderChoice);
                  console.log('removing defender choice from pool', {idx, defenderChoice, defenderPool: defenderPool.value});
                  if (idx !== -1) defenderPool.value.splice(idx, 1);
                  console.log('removing defender choice from pool', {defenderChoice, defenderPool: defenderPool.value});
                }

                // temporarily remove attacker options
                attackerOptions.forEach((team) => {
                  const idx = attackerPool.value.indexOf(team);
                  if (idx !== -1) attackerPool.value.splice(idx, 1);
                });

                // return the unselected attacker team
                if (attackerSelected) {
                  const other = attackerOptions.find((t) => t !== attackerSelected);
                  if (other) attackerPool.value.push(other);
                  const idx2 = attackerPool.value.indexOf(attackerSelected);
                  if (idx2 !== -1) attackerPool.value.splice(idx2, 1);
                }

                // remove map & mission
                if (mapChoice) {
                  const idx = mapsPool.value.indexOf(mapChoice);
                  if (idx !== -1) mapsPool.value.splice(idx, 1);
                }
                if (missionChoice) {
                  const idx = missionsPool.value.indexOf(missionChoice);
                  if (idx !== -1) missionsPool.value.splice(idx, 1);
                }

                // Step 3 extra ban
                if (step === 3) {
                  const extra = pairingData['extraBan3'];
                  if (extra) {
                    missionsPool.value = missionsPool.value.filter((m) => m !== extra);
                  }
                }
                // console.log('nextPairing before swap', { step, defenderChoice, attackerOptions, attackerSelected, mapChoice, missionChoice });
                // swap which side is Defender for the next pairing while preserving
                // the current remaining pools so previously-removed teams aren't reset
                // console.log('poolA/poolB', {poolA: JSON.parse(JSON.stringify(poolA.value)), poolB: JSON.parse(JSON.stringify(poolB.value))});
                const prevA = poolA.value.slice();
                const prevB = poolB.value.slice();
                // console.log('prevA/prevB', {prevA, prevB});
                // TUTAJ
                savedPools.a = prevB;
                savedPools.b = prevA;
                // savedPools.a = prevA;
                // savedPools.b = prevB;
                preservePools.value = true;
                defenderSide.value = defenderSide.value === 'our' ? 'enemy' : 'our';
                // console.log('nextPairing after swap', {savedPools,defenderSide});
              }

              pairingStep.value++;
            }

            function prevPairing() {
              if (pairingStep.value > 0) pairingStep.value--;
            }


    return {
      ourText,
      enemyText,
      mapsText,
      ourTeams,
      enemyTeams,
      maps,
      matrix,
      missions,
      generateMatrix,
      // matrix helper
      columnSums,
      // selection state
      defenderSide,
      bannedA,
      bannedB,
      pairingStep,
      poolA,
      poolB,
      mapsPool,
      missionsPool,
      pairingData,
      nextPairing,
      prevPairing,
      applyBans,
      resetPools,
      // suggestion helpers
      recommendEnemy,
      recommendMap,
      recommendMission,
      suggestedMissionBan,
      teamForLetter,
      // isOurTurn,
      sideForRole,
      suggestDefenderChoice,
      suggestAttackerSelection,
    };
  },
};
</script>

<style scoped>
textarea {
  width: 100%;
}
.input-row {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}
.input-block {
  flex: 1;
}
.matrix {
  margin-top: 1rem;
}
table {
  border-collapse: collapse;
  width: 100%;
  table-layout: fixed;
}
table td:nth-child(n+1):nth-child(-n+6) {
  background-color: #d4d4d4;
}

table th:nth-child(n+2):nth-child(-n+6) {
  background-color: #d4d4d4;
}

table td:nth-child(n+12) {
  background-color: #d4d4d4;
}

table th:nth-child(n+12) {
  background-color: #d4d4d4;
}

th,
td {
  padding: 0.25rem 0.5rem;
  word-break: break-word;
}

tfoot th,
tfoot td {
  font-weight: bold;
  background: #f0f0f0;
}
input {
  width: 100%;
  box-sizing: border-box;
}
select {
  padding: 3px;
}
.pairing-step {
  border: 1px solid black;
  padding: 0em 1em 1em 1em
}

.pairing-actions {
  display: grid;
  grid-template-columns: auto auto auto;
  gap: 1rem;
  margin-bottom: 1rem;
}

select::-webkit-scrollbar {
  width: 1px;
}
</style>
