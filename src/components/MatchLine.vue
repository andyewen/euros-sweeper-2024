<template>
  <div
    class="match"
    :class="{
      'in-progress': matchInProgress,
      completed: matchCompleted,
    }"
  >
    <TeamBadge
      :team="teamsByCode[match.homeTeam.countryCode]"
      :person="peopleByTeamCode[match.homeTeam.countryCode]"
      class="team-spaced"
    />
    <span v-if="showScores" class="score">
      {{ match.score.regular.home }}
    </span>
    <template v-if="showScores">–</template><template v-else>vs</template>
    <span v-if="showScores" class="score">
      {{ match.score.regular.away }}
    </span>
    <TeamBadge
      :team="teamsByCode[match.awayTeam.countryCode]"
      :person="peopleByTeamCode[match.awayTeam.countryCode]"
      class="team-spaced"
    />
    <div v-if="showPenalties">
      <small class="in-progress-penalties">
        <strong
          >{{ match.score.penalty.home }} –
          {{ match.score.penalty.away }}</strong
        >
        on penalties
      </small>
    </div>
    <div class="match-kickoff">
      <small>{{ formattedKickOffTime }}</small>
    </div>
  </div>
</template>

<script>
import TeamBadge from "./TeamBadge.vue";

import { pad2 } from "../utils.js";

export default {
  components: {
    TeamBadge,
  },
  props: {
    match: Object,
    teamsByCode: Object,
    peopleByTeamCode: Object,
  },
  data() {
    return {};
  },
  computed: {
    matchInProgress() { return this.match.status === 'LIVE' },
    matchCompleted() { return this.match.status === 'FINISHED' },
    showScores() { return this.matchInProgress || this.matchCompleted },
    showPenalties() {
      return this.showScores && Object.hasOwn(this.match.score, 'penalty');
    },
    formattedKickOffTime() {
      const months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
      ];
      const t = new Date(this.match.kickOffTime.dateTime);
      
      const date = `${months[t.getMonth()]} ${t.getDate()}`;
      const time = `${pad2(t.getHours())}:${pad2(t.getMinutes())}`
      const tzOffsetInMinutes = -t.getTimezoneOffset();
      const tzOffset = 
        'GMT '
        + (tzOffsetInMinutes >= 0 ? "+" : "-")
        + pad2(Math.floor(Math.abs(tzOffsetInMinutes) / 60))
        + ':'
        + pad2(Math.abs(tzOffsetInMinutes) % 60);

      return `${date} ${time} (${tzOffset})`;
    },
  },
};
</script>
