# Review resolution contract

Repository: Saber5656/glossary; PR #1

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTNkDz86QDL_u

### Use the built CLI in the dogfood npm script

**Normative resolution**

dogfood scriptはpackage自身のbin PATHに依存せず、確実に存在するnode dist/cli/main.js等のlocal build entrypointを呼ぶ。script開始前にbuild artifactの存在/版本を検査し、未buildなら明示的にfailする。

**Focused verification before resolving this thread:**

fresh clone/CIでnpm install後にdogfoodを実行し、global npm linkなしでlocal CLIが起動すること、dist欠損時のerrorが明確であることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_v

### Let validate own config parsing

**Normative resolution**

validateはneedsConfig=trueでtop-level E_CONFIG_INVALIDへ短絡せず、schema/parse errorをV_SCHEMA findingとしてJSON結果へ含め、許可されたstore validationを継続する。実行に必須な破損だけは明示的fatalとして分類する。

**Focused verification before resolving this thread:**

unknown key、syntax error、valid configとstore errorを組み合わせ、validate --jsonのfinding shape、exit code、継続対象が契約どおりになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_w

### Scope URL scans to actual references

**Normative resolution**

site hardening scanはrendered HTMLのhref/src/action/style等のURL-bearing attribute、resource、script destinationを検査し、escaped glossary definition text内のliteral URLを外部referenceとして誤検出しない。危険なattribute URLは引き続きfailする。

**Focused verification before resolving this thread:**

definition本文にhttps://evil.exampleを含むfixtureとhref/srcへ同URLを置くfixtureをbuildし、前者はcontentとしてescapeされ通過、後者はcritical findingになることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_x

### Check path containment before reading files

**Normative resolution**

include/exclude globを展開した候補はstat/read/size sniff前にlexicalとrealpath containmentでrepoRoot内へ制限する。symlink ancestor、../、absolute patternは拒否または安全にskipし、外部fileのbytesを一度も読まない。

**Focused verification before resolving this thread:**

repo内通常file、../、absolute、repo内から外部へのsymlinkをfixture化し、外部fileのread counterが0で、scannerが安全なdiagnosticを返すことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_y

### Preserve identifier reference counts through merge

**Normative resolution**

RawCandidateはidentifier declarationのreference countを保持し、mergeのoccurrences/minOccurrences判定へ引き継ぐ。declaration一件でも実参照が閾値を満たすtermはE2後に失われない。

**Focused verification before resolving this thread:**

TS/Python fixtureで一つのdeclarationと複数referenceを抽出し、raw count・merged occurrences・minOccurrences判定が一致すること、少数参照termは正しく除外されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_z

### Carry raw identifier names as real aliases

**Normative resolution**

RawCandidateへraw identifier alias/surfaceを明示的に保持し、termKey groupingとは別にsurfaces/approve aliasesへ原形を出力する。PaymentReservationとpayment reservationが意図したtermへmergeされ、raw nameがsnippetだけに埋もれない。

**Focused verification before resolving this thread:**

camelCase、PascalCase、snake_case、自然語のfixtureをmergeし、canonical key、surfaces、aliasesが規定どおり保存され、承認/検索でraw identifierが参照可能なことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_1

### Pass MiniSearch load options used at build time

**Normative resolution**

miniSearchOptions()をbuild/loadで共有し、fields、storeFields、idField等のindex-shape optionsをserialized index作成時とloadJSON時に同一値で渡す。option version mismatchはsafe errorにする。

**Focused verification before resolving this thread:**

buildしたsearch-index.jsonをbrowser loaderで読み、検索結果とstore fieldsが復元されること、optionを変更したindexがsilent emptyにならず明示的に拒否されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_2

### Build the Git dependency before invoking npx

**Normative resolution**

Git dependency installでdistを生成するprepare/build hookまたは配布artifactを定義し、Pages workflowのnpx glossary build前に確実なbuild stepを置く。source installとregistry installの両方のentrypointを固定する。

**Focused verification before resolving this thread:**

npm install github:...をclean workspaceで実行しdist/cli/main.jsとbinが存在すること、続くnpx glossary build --out _siteがglobal installなしで成功することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_4

### Block lifecycle scripts before npm ci executes them

**Normative resolution**

CIはnpm ci --ignore-scriptsで依存を展開してからlockfile/package metadataのlifecycle scriptを検査し、許可判断後に必要なbuild scriptだけを明示実行する。検査がinstall後ではsupply-chain gateにならない。

**Focused verification before resolving this thread:**

postinstall/prepareがネットワークやfile writeを行うfixture dependencyでCIを実行し、ignore-scripts段階では実行されず、禁止scriptがgateでfailし、許可されたbuildだけが後段で動くことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_6

### Add a DOM test dependency for the client suite

**Normative resolution**

client testをjsdom environmentで動かすため、jsdomをdevDependenciesとlockfileへ追加し、Vitest configのenvironmentとpackage scriptを同一契約にする。browser-only suiteがdependency欠如でskipされない。

**Focused verification before resolving this thread:**

clean npm ci後にDOM suiteを実行しjsdomが解決されること、document/window依存testが実行された証拠を出すこと、dependencyを除くとCIがfailすることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_8

### Harden export output paths against symlink escapes

**Normative resolution**

GLOSSARY.mdのexport pathはrepoRoot内のlexical pathだけでなく既存ancestorのlstat/realpathを検査し、symlink経由の外部writeを拒否する。atomic rename先も同じcontainment policyを通す。

**Focused verification before resolving this thread:**

通常dir、repo内symlink→外部、途中ancestor symlink、既存fileをfixture化し、通常だけwrite成功、symlink escapeはwrite前にfailして外部fileが不変であることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDL_-

### Reference the Python fixture identifier enough times

**Normative resolution**

E2 fixtureのcreate_payment_reservationはextract.minOccurrences=3を満たすraw referencesを実際に含める。fixtureの宣言/docstringだけでfilterされないよう、期待するidentifierの参照数をfixture lintで固定する。

**Focused verification before resolving this thread:**

fixture extraction結果にcreate_payment_reservationが出現し、raw reference countが3以上、merge後もcandidateが残って期待goldenへ入ることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDMAA

### Include hostile files in the default scan

**Normative resolution**

hostile e2eの対象big/huge.txtとbin/blob.datをdefault include globが実際に列挙できる場所/extensionへ置くか、test configで明示includeする。size/binary skipのassertはenumeration結果と同じfixtureを対象にする。

**Focused verification before resolving this thread:**

init直後のdefault configでhostile fixtureをextractし、huge/binary skip countersが0でないこと、通常file extractionとの境界、config変更時のexpected behaviorを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDMAB

### Seed XSS candidates that extraction can actually find

**Normative resolution**

XSS e2e fixtureは実際のextractorが候補として出せるpatternを使い、curation scriptがそのraw keyをapproveできるようにする。任意HTML literalをcandidate keyと仮定せず、renderer escapingのassertを実際のterm pathへ結びつける。

**Focused verification before resolving this thread:**

extract→list candidates→approve→buildをclean fixtureで通し、両XSS candidateがE1/E3/E4のいずれかで発見され、rendered siteでscript/attribute injectionにならないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDMAD

### Reject duplicate approve keys before writes

**Normative resolution**

approve commandはnormalized keyをpreflightでdeduplicateし、同一keyが複数指定されたらwrite開始前にerrorを返す。all-or-nothing contractをtransaction/temp file swapで維持し、partial stateを残さない。

**Focused verification before resolving this thread:**

approve keyを一度・重複・case/normalization同値で指定し、重複はfilesystem/DB変更なしでfailし、異なるvalid keysだけが一括commitされることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDMAF

### Use JavaScript-compatible case-insensitive regex

**Normative resolution**

redact regexはJavaScriptで有効なi flagまたはsupported noncapturing modifierを使い、PCRE専用のinline (?i)を使用しない。実行時compile errorを起こさず、対象secret patternを同じcase policyでmaskする。

**Focused verification before resolving this thread:**

NodeのRegExp compile testでpatternが有効なことを確認し、API_KEY/Bearer等の大小文字variantをredactして、非対象文字列を誤maskしないことを検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkDz86QDMAG

### Detect bold lead facts before stripping Markdown markers

**Normative resolution**

boldLead fact extractorはmdastのstrong node/raw markdownをmarkers除去前に参照し、**売上確定**のようなlead factを正しく識別する。plain list textとstrong nodeの両入力で同じsemantic resultを定義する。

**Focused verification before resolving this thread:**

bold lead、plain lead、nested emphasis、複数list itemをfixture化し、strong nodeだけがboldLeadとして抽出され、E4 candidate/acceptance assertionへ届くことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
