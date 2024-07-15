<template>
  <div class="container">
    <h1>Euros 2024 Sweepstake</h1>

    <h2 class="match-header">Matches</h2>
    <div class="match-day-selector">
      <button
        @click="selectDate(addDays(this.selectedDate, -1))"
        :disabled="!canSelectDate(addDays(this.selectedDate, -1))"
      >
        &lt;
      </button>
      <div class="date" :class="{ 'is-today': selectedDate == today }">
        <div class="month">{{ formatMonth(selectedDate) }}</div>
        <div class="day">{{ formatDay(selectedDate) }}</div>
      </div>
      <button
        @click="selectDate(addDays(this.selectedDate, 1))"
        :disabled="!canSelectDate(addDays(this.selectedDate, 1))"
      >
        &gt;
      </button>
    </div>
    <div class="matches-container">
      <template
        v-if="matchesGroupedByDate?.[selectedDate]"
      >
        <MatchLine
          v-for="match in matchesGroupedByDate[selectedDate]"
          :match="match"
          :teamsByCode="teamsByCode"
        />
      </template>
      <template v-else-if="!matchesGroupedByDate"> loading... </template>
      <template v-else> there are no matches on the selected day! </template>
    </div>

    <h2>Players</h2>
    <table class="player-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Teams</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="person in people"
          :key="person.name"
        >
          <td class="person-name">{{ person.name }}</td>
          <td>
            <TeamBadge
              v-for="team in person.teams"
              :key="team.code"
              :team="team"
              :showPerson="false"
              class="team-spaced"
            />
          </td>
        </tr>
      </tbody>
    </table>
    <div v-if="unallocatedTeams?.length" class="unallocated-teams">
      <TeamBadge
        v-for="team in unallocatedTeams"
        :key="team.code"
        :team="team"
        :showPerson="false"
        class="team-spaced"
      />
    </div>

    
    <h2 class="groups-header">Groups</h2>
    <div>
      <small>Note: Groups are probably not ordered correctly if they are tied on points.</small>
    </div>
    <GroupTable
      v-for="group in groups"
      :key="group.code"
      :group="group"
    />

    <p><strong>Good luck!</strong> 🤑</p>
    <p class="credits">
      Made by Andrew Ewen
      <a
        href="https://github.com/andyewen/euros-sweeper-2024"
        class="source-link"
      >
        [source]
      </a>
    </p>
  </div>
</template>

<script>
import GroupTable from "./components/GroupTable.vue";
import MatchLine from "./components/MatchLine.vue";
import TeamBadge from "./components/TeamBadge.vue";

import { dateString, addDays, pad2, product } from "./utils.js";

import { teams as rawTeams } from './assets/teams.json';
import { people as rawPeople } from './assets/people.json';
import { getTransitionRawChildren } from "vue";
// import exampleMatches from './assets/example_matches.json';


class Team {
  constructor(code, name, groupCode) {
    this.code = code;
    this.name = name;
    this.groupCode = groupCode;

    this.person = null;
    this.group = null;
    this.groupStats = null;
    this.tournamentStats = null;
    this.knockedOut = false;
  }
}


class TeamStats {
  constructor() {
    this.wins = 0;
    this.losses = 0;
    this.draws = 0;
    this.goalsScored = 0;
    this.goalsConceded = 0;
  }
  getGoalDifference() {
    return this.goalsScored - this.goalsConceded;
  }
  getPoints() {
    return this.wins * 3 + this.draws;
  }
}


export default {
  components: {
    GroupTable,
    MatchLine,
    TeamBadge,
  },
  data() {
    const today = dateString(new Date());

    const teams = rawTeams.map((rawTeam) => new Team(rawTeam.code, rawTeam.name, rawTeam.group));

    const people = [];
    for (const rawPerson of rawPeople) {
      const person = {
        name: rawPerson.name,
        teams: teams
          .filter((t) => rawPerson.teamCodes.includes(t.code))
          .sort((teamA, teamB) => {
            if (teamA.name < teamB.name) return -1;
            if (teamA.name > teamB.name) return 1;
            return 0;
          }),
      };
      for (const team of person.teams) {
        team.person = person;
      }
      people.push(person);
    }

    const groups = Object
      .entries(Object.groupBy(teams, (t) => t.groupCode))
      .map(([groupCode, teams]) => ({ code: groupCode, teams }))
      .sort((a, b) => {
        if (a.code < b.code) return -1;
        if (a.code > b.code) return 1;
        return 0;
      });
    for (const group of groups) {
      for (const team of group.teams) {
        team.group = group;
      }
    }

    return {
      matches: null,
      today: today,
      selectedDate: today,
      teams,
      people,
      groups,
    };
  },
  created() {
    this.fetchMatches();
  },
  methods: {
    addDays,
    async fetchMatches() {
      const response = await fetch("https://match.uefa.com/v5/matches?competitionId=3&limit=100&offset=0&phase=TOURNAMENT&seasonYear=2024&order=ASC");
      const matches = await response.json();
      // const matches = exampleMatches;
      this.loadMatches(matches);
    },
    loadMatches(matches) {
      this.matches = matches.sort((a, b) => {
        a = new Date(a.kickOffTime.dateTime);
        b = new Date(b.kickOffTime.dateTime);
        if (a < b) return -1;
        if (a > b) return 1;
        return 0;
      });

      this.generateTeamStats(this.matches);
      this.generateTeamsKnockedOut(this.matches);
      this.sortGroupTables();
    },
    generateTeamStats(matches) {
      for (const team of this.teams) {
        team.groupStats = new TeamStats();
        team.tournamentStats = new TeamStats();
      }

      matches = matches.filter((match) => match.winner);
      for (const match of matches) {
        let winnerCode = null;
        if (match.winner.match.reason.startsWith('WIN')) {
          winnerCode = match.winner.match.team.countryCode;
        }

        const homeTeam = this.teamsByCode[match.homeTeam.countryCode];
        const awayTeam = this.teamsByCode[match.awayTeam.countryCode]

        const statsKeysToUpdate = ['tournamentStats'];
        if (match.round.mode === 'GROUP') {
          statsKeysToUpdate.push('groupStats');
        }

        for (const statsKey of statsKeysToUpdate) {
          if (winnerCode === homeTeam.code) {
            homeTeam[statsKey].wins++;
            awayTeam[statsKey].losses++;
          } else if (winnerCode === awayTeam.code) {
            awayTeam[statsKey].wins++;
            homeTeam[statsKey].losses++;
          } else {
            homeTeam[statsKey].draws++;
            awayTeam[statsKey].draws++;
          }

          homeTeam[statsKey].goalsScored += match.score.total.home;
          homeTeam[statsKey].goalsConceded += match.score.total.away;
          awayTeam[statsKey].goalsScored += match.score.total.away;
          awayTeam[statsKey].goalsConceded += match.score.total.home;
        }
      }
    },
    generateTeamsKnockedOut(matches) {
      const groupMatches = matches.filter((match) => match.round.mode === 'GROUP');
      const knockoutMatches = matches.filter(
        (match) => ['KNOCKOUT', 'FINAL'].includes(match.round.mode)
      );

      const groupMatchesComplete = groupMatches.every((match) => match.status === 'FINISHED');
      if (!groupMatchesComplete) {
        return;
      }

      const remainingTeams = new Set(knockoutMatches
        .map((match) => [match.homeTeam.countryCode, match.awayTeam.countryCode])
        .flat()
        .filter((teamCode) => teamCode),
      );
      for (const match of knockoutMatches) {
        if (!match.winner?.match.reason.startsWith('WIN')) continue;

        const winnerCode = match.winner.match.team.countryCode;
        const loserCode = (
          winnerCode === match.homeTeam.countryCode ? 
          match.awayTeam.countryCode : match.homeTeam.countryCode
        );
        remainingTeams.delete(loserCode);
      }

      for (const team of this.teams) {
        team.knockedOut = !remainingTeams.has(team.code);
      }

      for (const person of this.people) {
        person.teams.sort((teamA, teamB) => teamA.knockedOut - teamB.knockedOut);
      }
    },
    sortGroupTables() {
      for (const group of this.groups) {
        group.teams.sort((teamA, teamB) => {
          const teamAPoints = teamA.groupStats.getPoints();
          const teamBPoints = teamB.groupStats.getPoints();
          if (teamAPoints !== teamBPoints) return teamBPoints - teamAPoints;
          return 0;
        });
      }
    },
    formatDay(d) {
      return pad2(new Date(d).getDate());
    },
    formatMonth(d) {
      return [
        "JAN",
        "FEB",
        "MAR",
        "APR",
        "MAY",
        "JUN",
        "JUL",
        "AUG",
        "SEP",
        "OCT",
        "NOV",
        "DEC",
      ][new Date(d).getMonth()];
    },
    canSelectDate(date) {
      if (!this.dateRange) return false;
      return date >= this.dateRange.min && date <= this.dateRange.max;
    },
    selectDate(date) {
      if (this.canSelectDate(date)) {
        this.selectedDate = date;
      }
    },
    
  },
  computed: {
    teamsByCode() {
      return Object.fromEntries(this.teams.map((t) => [t.code, t]));
    },
    unallocatedTeams() {
      return this.teams
        .filter((t) => t.person == null)
        .sort((teamA, teamB) => {
          if (teamA.name < teamB.name) return -1;
          if (teamA.name > teamB.name) return 1;
          return 0;
        });
    },
    matchesGroupedByDate() {
      if (!this.matches) return null;
      return Object.groupBy(this.matches, (m) => m.kickOffTime.date);
    },
    dateRange() {
      if (!this.matchesGroupedByDate || !Object.keys(this.matchesGroupedByDate).length) return null;
      const matchDates = Object.keys(this.matchesGroupedByDate);
      return { min: matchDates.at(0), max: matchDates.at(-1) };
    },
    loadedMatches() {
      return Boolean(this.matches);
    },
  },
  watch: {
    dateRange() {
      if (!this.dateRange) return null;
      // Clamp selectedDate when dateRange changes.
      if (this.selectedDate < this.dateRange.min) this.selectedDate = this.dateRange.min;
      if (this.selectedDate > this.dateRange.max) this.selectedDate = this.dateRange.max;
    },
  }
};
</script>

<style>
body {
  background-color: #111;
  font-family: sans-serif;
  color: #ccc;
  touch-action: manipulation;
}
.container {
  max-width: 900px;
  margin: 0 auto;
}

a:link {
  color: #80d0ff;
}
a:visited {
  color: #39647d;
}
a:hover {
  color: #b8e5ff;
}

.match-day-selector .date {
  display: inline-block;
  text-align: center;
  padding: 0 4px;
}

.match-day-selector .date.is-today {
  font-weight: bold;
}

.match-day-selector button {
  vertical-align: top;
  background-color: #111;
  color: #ccc;
  height: 36px;
  width: 68px;
  border: 1px solid #ccc;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  cursor: pointer;
}

.match-day-selector button:focus {
  outline: none;
  box-shadow: 0px 0px 2px 1px #999;
}

.match-day-selector button:active {
  outline: none;
  box-shadow: 0px 0px 4px 2px #999;
}

.match-day-selector button:hover {
  background-color: #222;
}

.match-day-selector button:disabled {
  background-color: #111;
  border-color: #333;
  color: #333;
  cursor: not-allowed;
}

.matches-container {
  padding-top: 8px;
}

h2 {
  margin-bottom: 8px;
}

.match-header {
  margin-top: 0;
}

.match {
  margin-bottom: 20px;
  border-left: 6px solid #333;
  padding-left: 8px;
}

@keyframes in-progress-flash {
  from {
    border-left-color: #333;
  }
  to {
    border-left-color: #ccc;
  }
}

.match.in-progress {
  animation-name: in-progress-flash;
  animation-duration: 1s;
  animation-direction: alternate;
  animation-iteration-count: infinite;
  animation-timing-function: ease-out;
}

.match.completed {
  border-left-color: #ccc;
}

.match .team {
  margin-left: 0;
  margin-right: 0;
}

.match .score {
  font-weight: bold;
}

table {
  border-collapse: collapse;
}

table.player-table {
  width: 100%;
}

table.group-table {
  width: 100%;
}

.person-row-knocked-out {
  color: #999;
}

.person-row-knocked-out .person-name {
  text-decoration: line-through;
}

.prize {
  font-weight: bold;
}

.final-place-1 .prize {
  color: gold;
}

.final-place-2 .prize {
  color: silver;
}

.final-place-3 .prize {
  color: #8a4c00;
}

@media only screen and (max-width: 500px) {
  .hide-small {
    display: none;
  }
}

.col2 {
  width: 50%;
  display: inline-block;
  box-sizing: border-box;
  padding: 8px;
  vertical-align: top;
}

.col2:blank {
  display: none;
}

@media only screen and (max-width: 990px) {
  .col2 {
    width: 100%;
    padding: 8px 0;
  }
}

th,
td {
  border: 1px solid #333;
  padding: 8px;
  text-align: left;
}

.unallocated-teams {
  margin-top: 16px;
}

.team {
  display: inline-block;
  background-color: #333;
  border-radius: 4px;
  padding: 4px;
  white-space: nowrap;
}

.team-spaced {
  margin: 4px;
}

.team.knocked-out {
  text-decoration: line-through;
  background-color: #222;
  color: #999;
}

.team .flag {
  /* height: 16px; */
  border-radius: 4px;
}
.team.knocked-out .flag {
  filter: brightness(25%);
}

.credits {
  font-size: 13.3333px;
}
.credits a {
  font-family: monospace;
}
</style>
