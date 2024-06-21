<template>
  <div class="team" :class="{ 'knocked-out': knockedOut }">
    <template v-if="team">
      {{ team.name }}
      <template v-if="showPerson">
        <template v-if="person"> ({{ person.name }}) </template>
        <template v-else> (N/A) </template>
      </template>
      <img :src="getFlagUrl(team.code)" height="16" class="flag" />
    </template>
    <template v-else> n/a </template>
  </div>
</template>

<script>
const fifaToIso = {
  QAT: "qa",
  ECU: "ec",
  SEN: "sn",
  NED: "nl",
  ENG: "gb-eng",
  IRN: "ir",
  USA: "us",
  WAL: "gb-wls",
  ARG: "ar",
  KSA: "sa",
  MEX: "mx",
  POL: "pl",
  FRA: "fr",
  AUS: "au",
  DEN: "dk",
  TUN: "tn",
  ESP: "es",
  CRC: "cr",
  GER: "de",
  JPN: "jp",
  BEL: "be",
  CAN: "ca",
  MAR: "ma",
  CRO: "hr",
  BRA: "br",
  SRB: "rs",
  SUI: "ch",
  CMR: "cm",
  POR: "pt",
  GHA: "gh",
  URU: "uy",
  KOR: "kr",
  SCO: "gb-sct",
  UKR: "ua",
  ROU: "ro",
  ALB: "al",
  HUN: "hu",
  GEO: "ge",
  CZE: "cz",
  ITA: "it",
  SVN: "si",
  SVK: "sk",
  TUR: "tr",
  AUT: "at",
};

function getFlagUrl(fifaCountryCode) {
  const isoCountryCode = fifaToIso[fifaCountryCode] ?? "xx";
  return `https://flagcdn.com/${isoCountryCode}.svg`;
}

export default {
  props: {
    team: Object,
    person: Object,
    showPerson: {
      type: Boolean,
      default: true,
    },
    knockedOut: {
      type: Boolean,
      default: false,
    },
  },
  methods: {
    getFlagUrl,
  },
};
</script>
