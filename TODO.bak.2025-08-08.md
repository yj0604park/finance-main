# TODO (Backup 2025-08-08)

- [x] Frontend: upgrade `@apollo/client` to 3.10.0 to satisfy GraphQL Code Generator suspense hooks (`SkipToken`, `skipToken`, `SuspenseQueryHookOptions`).
  - Command used: `npm i @apollo/client@3.10.0 --save-exact`
- [ ] Frontend build currently fails on unrelated TS path error:
  - TS2307: Cannot find module `src/models/bank`.
  - Action items:
    - Verify file exists (e.g., `src/models/bank.ts` or `bank/index.ts`).
    - If using path aliases like `src/*`, ensure `tsconfig.json` has `baseUrl`/`paths` and CRA is configured (e.g., `react-app-rewired`/`craco` or `tsconfig-paths` plugin) to respect them.
    - Adjust imports to relative paths if alias support is not enabled.
- [ ] Run `npm run build` again after fixing the import.
- [ ] (Optional) Run `npm audit fix` to address vulnerabilities reported by npm.