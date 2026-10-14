# Trenérský deník U8 – verze 3

Mobilní webová aplikace pro plánování a evidenci fotbalových tréninků.

## Co je nového
- měsíční kalendář s označením naplánovaných a dokončených tréninků
- výběr data a zobrazení všech tréninků v daný den
- připravená volitelná integrace se Supabase pro přihlášení e-mailem a cloudovou synchronizaci
- automatické odesílání změn do cloudu po přihlášení
- lokální export/import zálohy JSON stále zůstává dostupný
- ruční záloha a obnova dat přímo do soukromého GitHub repozitáře

## Záloha dat na GitHub

V části **Statistiky → Záloha na GitHub** lze ručně uložit nebo obnovit soubor se všemi tréninky a cvičeními.

1. Vytvoř samostatný **soukromý** GitHub repozitář, například `trenersky-denik-zalohy`. Nepoužívej veřejný repozitář aplikace pro osobní zálohy.
2. Na GitHubu otevři Settings → Developer settings → Personal access tokens → Fine-grained tokens a vytvoř token omezený pouze na tento repozitář.
3. Nastav Repository permissions → **Contents: Read and write**. Ostatní oprávnění ponech vypnutá.
4. V aplikaci vyplň GitHub uživatele, název repozitáře, větev (obvykle `main`) a cestu (výchozí `zaloha/trenersky-denik-u8.json`). Token vlož až těsně před zálohováním nebo obnovou.
5. Klikni na **Zálohovat data na GitHub**. Při obnově použij tlačítko **Obnovit data z GitHubu**; obnova přepíše aktuální data v aplikaci.

Token se ukládá jen do pole v paměti stránky a po akci se pole vymaže. Aplikace jej neukládá do `localStorage` ani do zdrojového kódu. V prohlížeči však token během akce musí být použit, proto jej vytvoř s minimálními oprávněními a pouze pro samostatný soukromý repozitář. Aplikace si zapamatuje jen název repozitáře a cestu, nikoli token. Záloha je ruční, nikoli automatická. GitHub vytvoří commit při každém přepsání souboru, takže starší verze lze dohledat v historii repozitáře.

Oficiální dokumentace: [GitHub REST API – repository contents](https://docs.github.com/en/rest/repos/contents) a [oprávnění fine-grained tokenů](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens).

## Důležité k synchronizaci
Cloudová synchronizace **není aktivní, dokud si nenastavíš vlastní projekt Supabase**. V aplikaci otevři Statistiky → Cloudová synchronizace a zadej Project URL a veřejný publishable/anon klíč. Používej pouze veřejný klíč, nikdy `service_role` nebo secret key.

### 1. Vytvoření projektu
1. Otevři https://supabase.com/ a vytvoř projekt.
2. V nastavení projektu zkopíruj Project URL a publishable key (nebo legacy anon public key).
3. V Supabase otevři SQL Editor a spusť následující SQL:

```sql
create table if not exists public.coach_u8_data (
  user_id uuid primary key references auth.users(id) on delete cascade,
  payload jsonb not null,
  updated_at timestamptz not null default now()
);

alter table public.coach_u8_data enable row level security;

create policy "Users can read own coach data"
on public.coach_u8_data for select
to authenticated
using (auth.uid() = user_id);

create policy "Users can insert own coach data"
on public.coach_u8_data for insert
to authenticated
with check (auth.uid() = user_id);

create policy "Users can update own coach data"
on public.coach_u8_data for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

### 2. Nastavení přihlášení
1. V Supabase otevři Authentication → URL Configuration.
2. Nastav Site URL na publikovanou HTTPS adresu aplikace.
3. Přidej stejnou adresu do Redirect URLs.
4. Pokud chceš přihlašování přes e-mailový odkaz, nastav SMTP / e-mailové doručování podle limitů projektu. Výchozí e-mailové doručování může mít omezení.
5. V aplikaci zadej URL a veřejný klíč, ulož nastavení, zadej e-mail a klikni na „Poslat přihlašovací odkaz“.

### 3. První synchronizace
- Po přihlášení se aplikace zeptá, zda načíst data z cloudu, nebo nahrát místní data.
- Pokud je to první použití, zvol nahrání místních dat do cloudu.
- Při dalších změnách se aplikace pokusí automaticky synchronizovat data.
- Na druhém zařízení otevři stejnou aplikaci, zadej stejnou Supabase URL a veřejný klíč a přihlas se stejným e-mailem.

**Pozor:** první synchronizace může nahradit data na jednom zařízení daty z druhého. Před prvním použitím cloudu udělej JSON export na všech zařízeních, kde už máš záznamy. Pokud se na více zařízeních upravují data současně, tato jednoduchá verze používá poslední zápis jako celek a neřeší slučování jednotlivých záznamů. Pro produkční použití je vhodné doplnit verzování a řešení konfliktů.

## Publikování přes GitHub Pages
1. Vytvoř nový repozitář například `trenersky-denik-u8`.
2. Nahraj všechny soubory z ZIPu přímo do kořenové složky.
3. Settings → Pages → Deploy from a branch → `main` → `/(root)`.
4. Po publikaci otevři HTTPS adresu v Safari na iPhonu.
5. Zvol Sdílet → Přidat na plochu.

## Co je součástí
- `index.html` – aplikace
- `manifest.webmanifest` – nastavení instalace na plochu
- `sw.js` – základní offline cache
- `icon-192.png`, `icon-512.png` – ikony
- `README.md` – tento návod

## Ukládání dat a soukromí
Bez cloudu se data ukládají pouze v localStorage daného prohlížeče. Cloudová data se ukládají do tabulky `coach_u8_data` pod identitou přihlášeného uživatele. Row Level Security (RLS) omezuje čtení a zápis na vlastní `user_id`. Nezadávej do aplikace tajné klíče.
