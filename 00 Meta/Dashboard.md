## Not started
```dataview
TABLE topic, area FROM "10 Java" OR "20 Spring"
WHERE status = "todo" SORT area, topic
```

## Shaky — drill these
```dataview
TABLE confidence, last-reviewed FROM ""
WHERE confidence <= 2 AND status != "todo" SORT confidence ASC
```

## Going stale
```dataview
TABLE last-reviewed FROM ""
WHERE status = "solid" AND last-reviewed <= date(today) - dur(21 days)
```

