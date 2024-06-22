<template>
  <div class="group-table-container col2">
    <table class="group-table">
      <thead>
        <tr>
          <th colspan="6">Group {{ group.code }}</th>
        </tr>
        <tr>
          <th>Team</th>
          <th class="hide-small">W</th>
          <th class="hide-small">D</th>
          <th class="hide-small">L</th>
          <th class="hide-small">GD</th>
          <th>Pts</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="team in group.teams" :key="team.code">
          <td>
            <TeamBadge
              :team="team"
              data-:knocked_out="team_knocked_out[team.code]"
              class="team-spaced"
            />
          </td>
          <td class="hide-small">
            {{ loadedMatches ? team.group_stats.wins : "?" }}
          </td>
          <td class="hide-small">
            {{ loadedMatches ? team.group_stats.draws : "?" }}
          </td>
          <td class="hide-small">
            {{ loadedMatches ? team.group_stats.losses : "?" }}
          </td>
          <td class="hide-small">
            <template v-if="loadedMatches">
              <template v-if="team.group_stats.get_goal_difference() >= 0"
                >+</template
              >{{ team.group_stats.get_goal_difference() }}
            </template>
            <template v-else> ? </template>
          </td>
          <td>{{ loadedMatches ? team.group_stats.get_points() : "?" }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import TeamBadge from "./TeamBadge.vue";

export default {
  components: {
    TeamBadge,
  },
  props: {
    group: Object,
    loadedMatches: Boolean,
  },
};
</script>
