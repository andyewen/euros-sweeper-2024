<template>
  <div
    class="match"
    :class="{
      'in-progress': match.status == 'in_progress',
      completed: match.status == 'completed',
    }"
  >
    <template v-if="match">
      <TeamBadge
        :team="teamsByCode[match.homeTeam.countryCode]"
        :person="peopleByTeamCode[match.homeTeam.countryCode]"
        class="team-spaced"
      />
      <span v-if="showScores" class="score">
        {{ match.home_team.goals }}
      </span>
      <template v-if="showScores">–</template><template v-else>vs</template>
      <span v-if="showScores" class="score">
        {{ match.away_team.goals }}
      </span>
      <TeamBadge
        :team="teamsByCode[match.awayTeam.countryCode]"
        :person="peopleByTeamCode[match.awayTeam.countryCode]"
        class="team-spaced"
      />
      <!-- <div v-if="show_penalties">
        <small class="in-progress-penalties">
          <strong
            >{{ match.home_team.penalties }} –
            {{ match.away_team.penalties }}</strong
          >
          on penalties
        </small>
      </div> -->
      <div class="match-kickoff">
        <small>{{ formattedKickOffTime }}</small>
      </div>
    </template>
    <p v-else>n/a</p>
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
    showScores: function () {
      return false;
      return (
        this.match.status == "in_progress" || this.match.status == "completed"
      );
    },
    // show_penalties: function () {
    //   return (
    //     this.show_scores &&
    //     this.match.home_team.penalties + this.match.away_team.penalties > 0
    //   );
    // },
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
