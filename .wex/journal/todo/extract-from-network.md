# Extract organizations / companies from network into symfony-company

Opened: 2026-09-24
Updated: 2026-09-24
Author: agent:archeology

## Read this first — status of this todo

> **This is a proposal for discussion, not an order to code.** It was written by the 2026-09 network archaeology pass. Read it, then discuss it with the owner: every design choice and recommendation below is to be challenged and validated **before** any code is written. Do not start implementing on your own.
>
> - Context: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/.wex/knowledge/readme/archeology/index.md.j2` (entry point, order between packages), then `sources.md.j2` (where the legacy code lives: archive repo, branch checkouts, GitLab issues) and the domain page linked below.
> - Pending owner decisions affecting this work are listed in `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/.wex/knowledge/readme/archeology/recap.md.j2`, section "Décisions qui t'attendent". Where this todo assumes an answer, treat it as an open question.
> - Safety: `NETWORK/local/network` runs on **production data** (real bookkeeping, real invoices in `var/`, a prod dump in `.wex/mysql/dumps/`) — read its code only, never run anything against it. Anonymize any fixture taken from network (bank exports, FEC, mails contain real names/accounts). Never copy secrets found in its history (Stripe keys, tokens, passwords, private keys).

## Goal

Give `wexample/symfony-company` (currently no code) its content: a generic, UUID-based model of **legal parties** (organizations: companies, associations, sole traders), their **legal forms**, **membership** of users with roles, an **organization voter**, the **"self" organization** provider, and the **country profile** system (FR + BE) that network built in 2026-07 for Belgian invoices. network 2027 will extend these classes with its app entity.

## Read first

- Knowledge page (data model, field semantics, seeds, pitfalls, recommended design): `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/.wex/knowledge/readme/archeology/organization.md.j2`
- Related: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/.wex/knowledge/readme/archeology/accounting.md.j2` (banks, invoices, VAT), `…/archeology/mail-notification.md.j2` (org as mail recipient), `…/archeology/user.md.j2` (user side of membership).
- Issues (`/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/gitlab/issues/`): `036.md` (org = invoice owner), `112.md` (labels), `121.md`, `122.md` (banks as orgs), `231.md`, `232.md`, `266.md` (rights), `267.md` (EI prefix), `289.md` (teams), `306.md` (business profile).
- Conventions: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/SERVICES/local/app-board`; run `wex ai::design/rules --formatter php-code` in this package; sibling layout example `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/PACKAGES/PHP/packages/wexample/symfony-geo` (bundle class, `DependencyInjection/`, `Resources/config/services.yaml`, composer `extra.install.bundle_env`).

## Prerequisites / dependencies

- `wexample/symfony-helpers` >= 9 (UUID v7 `AbstractEntity`, `HasEmailTrait`, `HasTitleTrait`…).
- `wexample/symfony-geo` for `Country` (and `AbstractAddress`, see `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/PACKAGES/PHP/packages/wexample/symfony-geo/.wex/journal/todo/extract-from-network-address.md` — do that first or in parallel).
- Must NOT depend on `symfony-accounting` (accounting depends on company, not the reverse).

## Decisions already implied by the owner

- Packages are the base, network 2027 rebuilds on them; PDFs are rendered by `SERVICES/local/pdf-factory` (the country profile must not pick PHP PDF page classes).
- IDs are UUIDs: no integer constants like `TYPE_ID_AUTO = 2` or `COUNTRY_ID_FRANCE = 37`; look up by code.
- Belgian support matters (last prod work, 2026-07).

## Steps

1. **Scaffold the bundle** (composer autoload `Wexample\SymfonyCompany\` → `src/`, require helpers + geo, `WexampleSymfonyCompanyBundle`, extension, services.yaml, `tests/` kernel like siblings). Acceptance: bundle boots in a test kernel.
2. **`LegalForm` entity** (code unique e.g. `frMicro2016`, title, countryCode ISO-2 nullable, `vatFranchise` bool, `displayPrefix` nullable e.g. `EI`, `active`). Provide a seed/data class with the network rows (1 `frMicro2016` Micro Entreprise après 2016 vatFranchise, 2 `frAutoPre2016` Auto Entreprise avant 2016 vatFranchise, 3 `frSas2017` SAS, 4 `frAsso1901` Association loi 1901, 5 `undefined` Inconnu, 6 `sarl` SARL) plus BE forms to confirm with the owner (SRL, SA, ASBL, personne physique). Replaces `hasVat()` hard-coded ids. Sources: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/src/Entity/OrganizationType.php`, `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/vendor/wexample/symfony-helpers/src/Entity/OrganizationType.php`. Test: seed idempotent, lookup by code.
3. **`AbstractOrganization` mapped superclass**: title (legal name), legalForm (ManyToOne, required), companyIdentifier, vatNumber (≤ 20), website, phone (was `number`), email (nullable, `#[Email]`), address (ManyToOne to the app's address class via resolve_target_entities, cascade persist, SET NULL), legalRepresentative, leader; methods `hasVat()` (= !legalForm.vatFranchise), `getCountry()` (address country), `getLegalRepresentativeName()` (legalRepresentative → leader → first owner member display name → title), `getDisplayName()` (prefix `EI` etc. from legal form, #267), `getCompanyIdentifierShort()` (SIREN = 9 first digits of SIRET). Sources: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/src/Entity/Organization.php` (best), `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Entity/Organization.php` (null-safe `getResponsibleMember`), `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Wex/BaseBundle/Entity/Organization.php`. Tests: each fallback branch of `getLegalRepresentativeName`, memberless org does not crash, `hasVat`.
4. **Membership**: `AbstractOrganizationMember` (organization, user via resolve_target_entities, role enum `owner|admin|member`, dateCreated) + repository helpers (`findMembership(user, org)`, `findOrganizationsOfUser`). Replaces `User.organization` (NOT NULL, cascade remove) + `User.organizationAccessLevel`. Provide `OrganizationMembershipService::createPersonalOrganization(user, legalFormCode='undefined')` equivalent of `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Service/EntityCrud/UserEntityCrudService.php::createUserOrganization` (title = username). Tests: add/remove member, deleting a user never deletes the organization.
5. **`OrganizationVoter`** with attributes `VIEW`, `EDIT`, `ADMINISTER`, `MANAGE_MEMBERS` resolved from membership role (+ configurable admin role bypass). Source of the bug to avoid: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/src/Voter/OrganizationVoter.php` (grants any ROLE_USER), `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Service/Entity/OrganizationEntityService.php::userCanAdministerOrganization`. Tests: matrix non-member / member / admin member / owner / platform admin × attributes (#232: a member can edit its own org).
6. **`SelfOrganizationProvider`** (replaces `CompanyService`): bundle config `wexample_symfony_company.self_organization_id` (UUID) → `getSelfOrganization()`. Source: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Service/CompanyService.php`. Test with config.
7. **Country profiles**: `Interface/CountryProfileInterface` (countryCode, companyIdentifierLabel, vatNumberLabel, `isCompanyIdentifierIncludedInVatNumber`, legal-mention translation keys by document kind — take an enum/string "document kind" + org instead of network's `Invoice`), `AbstractCountryProfile`, `DefaultCountryProfile` (= FR), `CountryProfileResolver` (tagged iterator `wexample_symfony_company.country_profile`, `resolve(?AbstractOrganization)`, `resolveByCode`), FR and BE profiles + translations (FR texts from `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/front/pdf/invoice.fr.yml` keys `legal.law_fr_l_123`, `law_fr_l_441`, `law_be_payment`). Sources: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/local/network/src/Country/` (all 6 files). Drop `getInvoicePageClass()` (pdf-factory concern). Tests: resolution by address country, fallback, BE identifier-in-VAT rule, FR no-VAT org gets L123 mention for bills.
8. **Identifier validators** (new, small): Symfony constraints `Siret` (14 digits + Luhn), `Siren`, `BelgianEnterpriseNumber` (10 digits, mod 97), `VatNumber` (country prefix format; FR key check), chosen per country profile. Tests with valid/invalid samples.
9. **Forms (optional, after symfony-forms conventions)**: organization main form (legalForm, title, leader, legalRepresentative, companyIdentifier, vatNumber, website, phone, email) with labels from the country profile (#112: "Raison sociale", "SIRET"). Source: `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/src/Form/Entity/Organization/OrganizationEntityForm.php`, labels `/home/weeger/Desktop/WIP/WEB/WEXAMPLE/NETWORK/archeo/trees/develop-131-fos-user/front/entity/organization.fr.yml`.
10. **Remove the legacy copy**: open a follow-up todo in `symfony-helpers` to delete `src/Entity/Organization.php` / `OrganizationType.php` once `symfony-accounting` points to company (coordinate; do not delete here).
11. README (usage: extend `AbstractOrganization`, register profiles) + knowledge pages.

## Do not

- Do not port banking (`isBank`, `accountingTransactions`, `accountingCode`, `FrBankInfo2018Trait`) — they stay in `symfony-accounting` (`AbstractBankOrganizationEntity`) or the app.
- Do not port `invoiceCodePrefix` / `InvoiceSequence` here (accounting, org-scoped numbering); at most leave an extension point.
- Do not port `projectContributors`, mail recipient helpers (`getMailableRecipients`), picture upload specifics (use symfony-file later), API Platform annotations, `buildAccountingAccountName` (accounting).
- Do not hard-code integer ids; do not reproduce `User.organization` cascade remove.
- Do not read or copy production data from `NETWORK/local/network/.wex/mysql/dumps` beyond the reference seeds listed here.

## Acceptance criteria

- Package tests (PHPUnit, test kernel with an in-memory/sqlite or test DB) cover: legal-form seed, organization fallbacks and `hasVat`, membership lifecycle, voter matrix, self-organization provider, country-profile resolution FR/BE/default, identifier validators.
- network's current behaviours listed in the knowledge page "How it works" are reproducible by extending the package classes (documented in README).
