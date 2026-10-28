<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem the reports show. `issues` maps a problem to its ticket.
  let { ticket, issues }: { ticket: number; issues: Record<string, string> } = $props();

  const all = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(all));

  // A date moved by one day is not a swap, whatever the tenant writes: it gets its own row.
  const problemOf = (r: Report) => (notes.oneDayMove.includes(r.id) ? 'one-day' : causeOf(r, tenantOf(r)) ?? 'unplaced');

  const groups = $derived(
    [...Map.groupBy(merged, problemOf).entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem,
          reports: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          channels: [...new Set(rs.map((r) => r.channel))].sort().join(', '),
          seen: `${days[0]} → ${days[days.length - 1]}`,
          mistagged: rs.filter((r) => r.meta.intakeTag !== problemOf(r)).length,
        };
      })
      .sort((a, b) => b.reports - a.reports),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ({all.length - merged.length} repeated in the export) ·
  <strong>{groups.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>intake tag differs</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code></td>
        <td>{g.reports}</td>
        <td>{g.tenants}</td>
        <td>{g.channels}</td>
        <td>{g.seen}</td>
        <td>{g.mistagged}</td>
        <td>{#if issues[g.problem]}<a href={issues[g.problem]}>{issues[g.problem].split('/').slice(-3).join('/').replace('/issues/', '#')}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
