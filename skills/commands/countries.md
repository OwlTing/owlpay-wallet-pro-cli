# Countries

List countries allowed for individual account registration. Used by `owlp onboard` to validate the user's country input during signup.

No authentication required.

## Commands

```bash
owlp countries --json
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp countries --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.country_code) \(.country_name)"'
```

## JSON Response

Wrapped in `{success, env, data}`. `data` is an array of countries where `is_individual_allow_register === true`:

```json
{
  "success": true,
  "env": "prod",
  "data": [
    {
      "country_code": "TW",
      "country_name": "Taiwan",
      "continent_name": "Asia"
    },
    {
      "country_code": "US",
      "country_name": "United States",
      "state_code": "CA",
      "state": "California",
      "continent_name": "North America"
    }
  ]
}
```

Countries with multi-region signup requirements have one row per state (with `state_code` + `state`). Other fields from the server's country response are passed through.

## Notes

- This is a reference list — not a status check. It does **not** tell you whether the current user is from an allowed country.
- Use this to populate a country picker or to validate user input before calling `owlp onboard`.
