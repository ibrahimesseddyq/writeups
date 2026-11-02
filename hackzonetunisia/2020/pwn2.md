# Pwn2

**CTF:** HackZoneTunisia 2020  
**Category:** misc  
**Original writeup:** <https://github.com/KEERRO/ctf-writeups/blob/master/Hack%20Zone%20Tunisia%202020/pwn/pwn2/sploit.py> — by **KEERRO**. All credit goes to the original author; this is an archived copy.

---

<head>

<meta charset="utf-8">

<link rel="dns-prefetch" href="https://github.githubassets.com">

<link rel="dns-prefetch" href="https://avatars.githubusercontent.com">

<link rel="dns-prefetch" href="https://github-cloud.s3.amazonaws.com">

<link rel="dns-prefetch" href="https://user-images.githubusercontent.com/">

<link rel="preconnect" href="https://github.githubassets.com" crossorigin>

<link rel="preconnect" href="https://avatars.githubusercontent.com">

<link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/light-fe3f886b577a.css" /><link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/dark-a1dbeda2886c.css" /><link data-color-theme="dark_dimmed" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/dark_dimmed-1ad5cf51dfeb.css" /><link data-color-theme="dark_high_contrast" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/dark_high_contrast-11d3505dc06a.css" /><link data-color-theme="dark_colorblind" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/dark_colorblind-8b800495504f.css" /><link data-color-theme="light_colorblind" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/light_colorblind-daa38c88b795.css" /><link data-color-theme="light_high_contrast" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/light_high_contrast-1b9ea565820a.css" /><link data-color-theme="light_tritanopia" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/light_tritanopia-e4be9332dd6c.css" /><link data-color-theme="dark_tritanopia" crossorigin="anonymous" media="all" rel="stylesheet" data-href="https://github.githubassets.com/assets/dark_tritanopia-0dcf95848dd5.css" />





<link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/primer-c581c4e461bb.css" />

<link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/global-0e278d45156f.css" />

<link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/github-dcaf0f44dbb1.css" />

<link crossorigin="anonymous" media="all" rel="stylesheet" href="https://github.githubassets.com/assets/code-26709f54a08d.css" />

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/wp-runtime-774bfe5ae983.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_stacktrace-parser_dist_stack-trace-parser_esm_js-node_modules_github_bro-327bbf-0aaeb22dd2a5.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/ui_packages_soft-nav_soft-nav_ts-21fc7a4a0e8f.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/environment-e059fd03252f.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_selector-observer_dist_index_esm_js-2646a2c533e3.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_delegated-events_dist_index_js-node_modules_github_details-dialog-elemen-63debe-c04540d458d4.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_relative-time-element_dist_index_js-b9368a9cb79e.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_fzy_js_index_js-node_modules_github_markdown-toolbar-element_dist_index_js-e3de700a4c9d.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_auto-complete-element_dist_index_js-node_modules_github_catalyst_-6afc16-e779583c369f.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_file-attachment-element_dist_index_js-node_modules_github_text-ex-3415a8-7ecc10fb88d0.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_filter-input-element_dist_index_js-node_modules_github_remote-inp-79182d-befd2b2f5880.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_primer_view-components_app_components_primer_primer_js-node_modules_gith-6a1af4-df3bc95b06d3.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/github-elements-fc0e0b89822a.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/element-registry-1641411db24a.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_lit-html_lit-html_js-9d9fe1859ce5.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_manuelpuyol_turbo_dist_turbo_es2017-esm_js-4140d67f0cc2.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_mini-throttle_dist_index_js-node_modules_github_alive-client_dist-bf5aa2-424aa982deef.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_primer_behaviors_dist_esm_dimensions_js-node_modules_github_hotkey_dist_-9fc4f4-d434ddaf3207.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_color-convert_index_js-35b3ae68c408.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_remote-form_dist_index_js-node_modules_github_session-resume_dist-def857-2a32d97c93c5.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_paste-markdown_dist_index_esm_js-node_modules_github_quote-select-15ddcc-1512e06cfee0.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_updatable-content_ts-430cacb5f7df.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_behaviors_keyboard-shortcuts-helper_ts-app_assets_modules_github_be-f5afdb-8dd5f026c5b9.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_sticky-scroll-into-view_ts-0af96d15a250.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_behaviors_include-fragment_ts-app_assets_modules_github_behaviors_r-4077b4-75370d1c1705.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_behaviors_commenting_edit_ts-app_assets_modules_github_behaviors_ht-83c235-7883159efa9e.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/behaviors-742151da9690.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_delegated-events_dist_index_js-node_modules_github_catalyst_lib_index_js-06ff531-32d7d1e94817.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/notifications-global-f5b58d24780b.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_morphdom_dist_morphdom-esm_js-node_modules_github_template-parts_lib_index_js-58417dae193c.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_virtualized-list_es_index_js-node_modules_github_memoize_dist_esm_index_js-8496b7c4b809.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_remote-form_dist_index_js-node_modules_delegated-events_dist_inde-70450e-0370b887db62.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/app_assets_modules_github_ref-selector_ts-7bdefeb88a1a.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/codespaces-d1ede1f1114e.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_filter-input-element_dist_index_js-node_modules_github_mini-throt-a33094-b03defd3289b.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_file-attachment-element_dist_index_js-node_modules_github_mini-th-85225b-226fc85f9b72.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/repositories-8093725f8825.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/topic-suggestions-7a1f0da7430a.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/code-menu-89d93a449480.js"></script>



<title>ctf-writeups/Hack Zone Tunisia 2020/pwn/pwn2 at master · KEERRO/ctf-writeups · GitHub</title>

<meta name="route-pattern" content="/:user_id/:repository/tree/*name(/*path)">



<meta name="current-catalog-service-hash" content="343cff545437bc2b0304c97517abf17bb80d9887520078e9757df416551ef5d6">

<meta name="request-id" content="C166:380B:4306931:448923E:6412213E" data-pjax-transient="true"/><meta name="html-safe-nonce" content="07acddf943169798734c20438cf2550bfab0963223e95a91e9fc1cc330bcd5af" data-pjax-transient="true"/><meta name="visitor-payload" content="eyJyZWZlcnJlciI6IiIsInJlcXVlc3RfaWQiOiJDMTY2OjM4MEI6NDMwNjkzMTo0NDg5MjNFOjY0MTIyMTNFIiwidmlzaXRvcl9pZCI6IjE4MDU5MzA1NDU4Nzc3NTQxNzQiLCJyZWdpb25fZWRnZSI6ImZyYSIsInJlZ2lvbl9yZW5kZXIiOiJmcmEifQ==" data-pjax-transient="true"/><meta name="visitor-hmac" content="94aa614f612e8e7444be0aa8d1638d69b3c7a08865d8835af686934e2aebc052" data-pjax-transient="true"/>

<meta name="hovercard-subject-tag" content="repository:162845937" data-turbo-transient>

<meta name="github-keyboard-shortcuts" content="repository,source-code,file-tree" data-turbo-transient="true" />



<meta name="selected-link" value="repo_source" data-turbo-transient>

<meta name="google-site-verification" content="c1kuD-K2HIVF635lypcsWPoD4kilo5-jA_wBFyT4uMY">

<meta name="google-site-verification" content="KT5gs8h0wvaagLKAVWq8bbeNwnZZK1r1XQysX3xurLU">

<meta name="google-site-verification" content="ZzhVyEFwb7w3e0-uOTltm8Jsck2F5StVihD0exw2fsA">

<meta name="google-site-verification" content="GXs5KoUUkNCoaAZn7wPN-t01Pywp9M3sEjnt_3_ZWPc">

<meta name="google-site-verification" content="Apib7-x98H0j5cPqHWwSMm6dNU4GmODRoqxLiDzdx9I">

<meta name="octolytics-url" content="https://collector.github.com/github/collect" />

<meta name="analytics-location" content="/<user-name>/<repo-name>/files/disambiguate" data-turbo-transient="true" />

<meta name="user-login" content="">

<meta name="viewport" content="width=device-width">



<meta name="description" content="Contribute to KEERRO/ctf-writeups development by creating an account on GitHub.">

<link rel="search" type="application/opensearchdescription+xml" href="/opensearch.xml" title="GitHub">

<link rel="fluid-icon" href="https://github.com/fluidicon.png" title="GitHub">

<meta property="fb:app_id" content="1401488693436528">

<meta name="apple-itunes-app" content="app-id=1477376905" />

<meta name="twitter:image:src" content="https://opengraph.githubassets.com/e68933ac284b7f665c8ea555e19d7e6118920c67571ea8ca1312106e9058e23e/KEERRO/ctf-writeups" /><meta name="twitter:site" content="@github" /><meta name="twitter:card" content="summary_large_image" /><meta name="twitter:title" content="ctf-writeups/Hack Zone Tunisia 2020/pwn/pwn2 at master · KEERRO/ctf-writeups" /><meta name="twitter:description" content="Contribute to KEERRO/ctf-writeups development by creating an account on GitHub." />

<meta property="og:image" content="https://opengraph.githubassets.com/e68933ac284b7f665c8ea555e19d7e6118920c67571ea8ca1312106e9058e23e/KEERRO/ctf-writeups" /><meta property="og:image:alt" content="Contribute to KEERRO/ctf-writeups development by creating an account on GitHub." /><meta property="og:image:width" content="1200" /><meta property="og:image:height" content="600" /><meta property="og:site_name" content="GitHub" /><meta property="og:type" content="object" /><meta property="og:title" content="ctf-writeups/Hack Zone Tunisia 2020/pwn/pwn2 at master · KEERRO/ctf-writeups" /><meta property="og:url" content="https://github.com/KEERRO/ctf-writeups" /><meta property="og:description" content="Contribute to KEERRO/ctf-writeups development by creating an account on GitHub." />



<link rel="assets" href="https://github.githubassets.com/">

<meta name="hostname" content="github.com">

<meta name="expected-hostname" content="github.com">

<meta name="enabled-features" content="TURBO_EXPERIMENT_RISKY,IMAGE_METRIC_TRACKING,GEOJSON_AZURE_MAPS">

<meta http-equiv="x-pjax-version" content="ef97471de14f8d2285f0269e8f0f7dc70845f693d3f6ccd2dd2daae5cd1bbebe" data-turbo-track="reload">

<meta http-equiv="x-pjax-csp-version" content="2a84822a832da97f1ea76cf989a357ec70c85713a2fd8f14c8421b76bbffe38c" data-turbo-track="reload">

<meta http-equiv="x-pjax-css-version" content="adfc12179419e463f9f320d07920b1684c9b7e060d4d9cd3a6cd5d0de37ce710" data-turbo-track="reload">

<meta http-equiv="x-pjax-js-version" content="711646ae23abb27cf728346f30f81c042d4428233a0795acf0e21ed664fe9d94" data-turbo-track="reload">

<meta name="turbo-cache-control" content="no-preview" data-turbo-transient="">

<meta data-hydrostats="publish">

<meta name="go-import" content="github.com/KEERRO/ctf-writeups git https://github.com/KEERRO/ctf-writeups.git">

<meta name="octolytics-dimension-user_id" content="46076094" /><meta name="octolytics-dimension-user_login" content="KEERRO" /><meta name="octolytics-dimension-repository_id" content="162845937" /><meta name="octolytics-dimension-repository_nwo" content="KEERRO/ctf-writeups" /><meta name="octolytics-dimension-repository_public" content="true" /><meta name="octolytics-dimension-repository_is_fork" content="false" /><meta name="octolytics-dimension-repository_network_root_id" content="162845937" /><meta name="octolytics-dimension-repository_network_root_nwo" content="KEERRO/ctf-writeups" />

<link rel="canonical" href="https://github.com/KEERRO/ctf-writeups/tree/master/Hack%20Zone%20Tunisia%202020/pwn/pwn2" data-turbo-transient>

<meta name="turbo-body-classes" content="logged-out env-production page-responsive">

<meta name="browser-stats-url" content="https://api.github.com/_private/browser/stats">

<meta name="browser-errors-url" content="https://api.github.com/_private/browser/errors">

<meta name="browser-optimizely-client-errors-url" content="https://api.github.com/_private/browser/optimizely_client/errors">

<link rel="mask-icon" href="https://github.githubassets.com/pinned-octocat.svg" color="#000000">

<link rel="alternate icon" class="js-site-favicon" type="image/png" href="https://github.githubassets.com/favicons/favicon.png">

<link rel="icon" class="js-site-favicon" type="image/svg+xml" href="https://github.githubassets.com/favicons/favicon.svg">

<meta name="theme-color" content="#1e2327">

<meta name="color-scheme" content="light dark" />

<link rel="manifest" href="/manifest.json" crossOrigin="use-credentials">

</head>

<body class="logged-out env-production page-responsive" style="word-wrap: break-word;">

<div data-turbo-body class="logged-out env-production page-responsive" style="word-wrap: break-word;">



<div class="position-relative js-header-wrapper ">

Skip to content

<span>

<span></span>

</span>



<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/vendors-node_modules_github_remote-form_dist_index_js-node_modules_delegated-events_dist_inde-94fd67-04fa93bb158a.js"></script>

<script crossorigin="anonymous" defer="defer" type="application/javascript" src="https://github.githubassets.com/assets/sessions-9920eaa99f50.js"></script>

<header class="Header-old header-logged-out js-details-container Details position-relative f4 py-3" role="banner">

<button type="button" class="Header-backdrop d-lg-none border-0 position-fixed top-0 left-0 width-full height-full js-details-target" aria-label="Toggle navigation">

<span>Toggle navigation</span>

</button>

<div class="container-xl d-flex flex-column flex-lg-row flex-items-center p-responsive height-full position-relative z-1">

<div class="d-flex flex-justify-between flex-items-center width-full width-lg-auto">



<svg height="32" aria-hidden="true" viewBox="0 0 16 16" version="1.1" width="32" data-view-component="true" class="octicon octicon-mark-github">

<path d="M8 0c4.42 0 8 3.58 8 8a8.013 8.013 0 0 1-5.45 7.59c-.4.08-.55-.17-.55-.38 0-.27.01-1.13.01-2.2 0-.75-.25-1.23-.54-1.48 1.78-.2 3.65-.88 3.65-3.95 0-.88-.31-1.59-.82-2.15.08-.2.36-1.02-.08-2.12 0 0-.67-.22-2.2.82-.64-.18-1.32-.27-2-.27-.68 0-1.36.09-2 .27-1.53-1.03-2.2-.82-2.2-.82-.44 1.1-.16 1.92-.08 2.12-.51.56-.82 1.28-.82 2.15 0 3.06 1.86 3.75 3.64 3.95-.23.2-.44.55-.51 1.07-.46.21-1.61.55-2.33-.66-.15-.24-.6-.83-1.23-.82-.67.01-.27.38.01.53.34.19.73.9.82 1.13.16.45.68 1.31 2.69.94 0 .67.01 1.3.01 1.49 0 .21-.15.45-.55.38A7.995 7.995 0 0 1 0 8c0-4.42 3.58-8 8-8Z"></path>

</svg>



<div class="flex-1">



Sign up



</div>

<div class="flex-1 flex-order-2 text-right">

<button aria-label="Toggle navigation" aria-expanded="false" type="button" data-view-component="true" class="js-details-target Button--link Button--medium Button d-lg-none color-fg-inherit p-1"> <span>

<span><div class="HeaderMenu-toggle-bar rounded my-1"></div>

<div class="HeaderMenu-toggle-bar rounded my-1"></div>

<div class="HeaderMenu-toggle-bar rounded my-1"></div></span>

</span>

</button>

</div>

</div>

<div class="HeaderMenu--logged-out p-responsive height-fit position-lg-relative d-lg-flex flex-column flex-auto pt-7 pb-4 top-0">

<div class="header-menu-wrapper d-flex flex-column flex-self-end flex-lg-row flex-justify-between flex-auto p-3 p-lg-0 rounded rounded-lg-0 mt-3 mt-lg-0">

<nav class="mt-0 px-3 px-lg-0 mb-3 mb-lg-0" aria-label="Global">



<div class="d-lg-flex flex-items-center px-3 px-lg-0 mb-3 mb-lg-0 text-center text-lg-left">

<div class="d-lg-flex min-width-0 mb-2 mb-lg-0">



<div class="header-search flex-auto position-relative js-site-search flex-self-stretch flex-md-self-auto mb-3 mb-md-0 mr-0 mr-md-3 scoped-search site-scoped-search js-jump-to"

>

<div class="position-relative">

</option></form><form class="js-site-search-form" role="search" aria-label="Site" data-scope-type="Repository" data-scope-id="162845937" data-scoped-search-url="/KEERRO/ctf-writeups/search" data-owner-scoped-search-url="/users/KEERRO/search" data-unscoped-search-url="/search" data-turbo="false" action="/KEERRO/ctf-writeups/search" accept-charset="UTF-8" method="get">

<label class="form-control header-search-wrapper input-sm p-0 js-chromeless-input-container header-search-wrapper-jump-to position-relative d-flex flex-justify-between flex-items-center">

<input type="text"

class="form-control js-site-search-focus header-search-input jump-to-field js-jump-to-field js-site-search-field is-clearable"

data-hotkey=s,/

name="q"



placeholder="Search"

data-unscoped-placeholder="Search GitHub"

data-scoped-placeholder="Search"

autocapitalize="off"

role="combobox"

aria-haspopup="listbox"

aria-expanded="false"

aria-autocomplete="list"

aria-controls="jump-to-results"

aria-label="Search"

data-jump-to-suggestions-path="/_graphql/GetSuggestedNavigationDestinations"

spellcheck="false"

autocomplete="off"

>

<input type="hidden" data-csrf="true" class="js-data-jump-to-suggestions-path-csrf" value="fA08NGCSMeZkEyXaGynZ36UsrNWCB+v4uIkJyf/wQyRuQM9GOdWRz/F+HXp5mdOuhfvgvg+YhVQfLzoseTO99g==" />

<input type="hidden" class="js-site-search-type-field" name="type" >

<svg xmlns="http://www.w3.org/2000/svg" width="22" height="20" aria-hidden="true" class="mr-1 header-search-key-slash"><path fill="none" stroke="#979A9C" opacity=".4" d="M3.5.5h12c1.7 0 3 1.3 3 3v13c0 1.7-1.3 3-3 3h-12c-1.7 0-3-1.3-3-3v-13c0-1.7 1.3-3 3-3z"></path><path fill="#979A9C" d="M11.8 6L8 15.1h-.9L10.8 6h1z"></path></svg>

<div class="Box position-absolute overflow-hidden d-none jump-to-suggestions js-jump-to-suggestions-container">









<span>No suggested jump to results</span>





</div>

</label>

</form> </div>

</div>

</div>

<div class="position-relative mr-lg-3 d-lg-inline-block">



Sign in



</div>



Sign up



</div>

</div>

</div>

</div>

</header>

</div>

<div id="start-of-content" class="show-on-focus"></div>

<div id="js-flash-container" data-turbo-replace>

<template class="js-flash-template">



<div class="flash flash-full {{ className }}">

<div class="px-2" >

<button autofocus class="flash-close js-flash-close" type="button" aria-label="Dismiss this message">

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-x">

<path d="M3.72 3.72a.75.75 0 0 1 1.06 0L8 6.94l3.22-3.22a.749.749 0 0 1 1.275.326.749.749 0 0 1-.215.734L9.06 8l3.22 3.22a.749.749 0 0 1-.326 1.275.749.749 0 0 1-.734-.215L8 9.06l-3.22 3.22a.751.751 0 0 1-1.042-.018.751.751 0 0 1-.018-1.042L6.94 8 3.72 4.78a.75.75 0 0 1 0-1.06Z"></path>

</svg>

</button>

<div aria-atomic="true" role="alert" class="js-flash-alert">



<div>{{ message }}</div>

</div>

</div>

</div>

</template>

</div>



<include-fragment class="js-notification-shelf-include-fragment" data-base-src="https://github.com/notifications/beta/shelf"></include-fragment>

<div

class="application-main "

data-commit-hovercards-enabled

data-discussion-hovercards-enabled

data-issue-and-pr-hovercards-enabled

>

<div itemscope itemtype="http://schema.org/SoftwareSourceCode" class="">

<main id="js-repo-pjax-container" >







<div id="repository-container-header" class="pt-3 hide-full-screen" style="background-color: var(--color-page-header-bg);" data-turbo-replace>

<div class="d-flex flex-wrap flex-justify-end mb-3 px-3 px-md-4 px-lg-5" style="gap: 1rem;">

<div class="flex-auto min-width-0 width-fit mr-3">



<div class=" d-flex flex-wrap flex-items-center wb-break-word f3 text-normal">

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-repo color-fg-muted mr-2">

<path d="M2 2.5A2.5 2.5 0 0 1 4.5 0h8.75a.75.75 0 0 1 .75.75v12.5a.75.75 0 0 1-.75.75h-2.5a.75.75 0 0 1 0-1.5h1.75v-2h-8a1 1 0 0 0-.714 1.7.75.75 0 1 1-1.072 1.05A2.495 2.495 0 0 1 2 11.5Zm10.5-1h-8a1 1 0 0 0-1 1v6.708A2.486 2.486 0 0 1 4.5 9h8ZM5 12.25a.25.25 0 0 1 .25-.25h3.5a.25.25 0 0 1 .25.25v3.25a.25.25 0 0 1-.4.2l-1.45-1.087a.249.249 0 0 0-.3 0L5.4 15.7a.25.25 0 0 1-.4-.2Z"></path>

</svg>



<span>



KEERRO

</span>

<span>/</span>



ctf-writeups



<span></span><span>Public</span>

</div>

</div>

</div>

<div id="responsive-meta-container" data-turbo-replace>

</div>

<nav data-pjax="#js-repo-pjax-container" aria-label="Repository" data-view-component="true" class="js-repo-nav js-sidenav-container-pjax js-responsive-underlinenav overflow-hidden UnderlineNav px-3 px-md-4 px-lg-5">

More

</div>

<turbo-frame id="repo-content-turbo-frame" target="_top" data-turbo-action="advance" class="">

<div id="repo-content-pjax-container" class="repository-content " >







<div class="clearfix container-xl px-3 px-md-4 px-lg-5 mt-4">

<div >



<div class="file-navigation mb-3 d-flex flex-items-start">



<div class="position-relative">

<details

class="js-branch-select-menu details-reset details-overlay mr-0 mb-0 "

id="branch-select-menu"

data-hydro-click-payload="{"event_type":"repository.click","payload":{"target":"REFS_SELECTOR_MENU","repository_id":162845937,"originating_url":"https://github.com/KEERRO/ctf-writeups/tree/master/Hack%20Zone%20Tunisia%202020/pwn/pwn2","user_id":null}}" data-hydro-click-hmac="b4e25b642492d7624326169a22cdcb094bde6e06643e82dbedcb0e808914df6e">

<summary class="btn css-truncate"

data-hotkey="w"

title="Switch branches or tags">

<svg text="gray" aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-git-branch">

<path d="M9.5 3.25a2.25 2.25 0 1 1 3 2.122V6A2.5 2.5 0 0 1 10 8.5H6a1 1 0 0 0-1 1v1.128a2.251 2.251 0 1 1-1.5 0V5.372a2.25 2.25 0 1 1 1.5 0v1.836A2.493 2.493 0 0 1 6 7h4a1 1 0 0 0 1-1v-.628A2.25 2.25 0 0 1 9.5 3.25Zm-6 0a.75.75 0 1 0 1.5 0 .75.75 0 0 0-1.5 0Zm8.25-.75a.75.75 0 1 0 0 1.5.75.75 0 0 0 0-1.5ZM4.25 12a.75.75 0 1 0 0 1.5.75.75 0 0 0 0-1.5Z"></path>

</svg>

<span>master</span>

<span></span>

</summary>



<div class="SelectMenu">

<div class="SelectMenu-modal">

<header class="SelectMenu-header">

<span>Switch branches/tags</span>

<button class="SelectMenu-closeButton" type="button" data-toggle-for="branch-select-menu"><svg aria-label="Close menu" aria-hidden="false" role="img" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-x">

<path d="M3.72 3.72a.75.75 0 0 1 1.06 0L8 6.94l3.22-3.22a.749.749 0 0 1 1.275.326.749.749 0 0 1-.215.734L9.06 8l3.22 3.22a.749.749 0 0 1-.326 1.275.749.749 0 0 1-.734-.215L8 9.06l-3.22 3.22a.751.751 0 0 1-1.042-.018.751.751 0 0 1-.018-1.042L6.94 8 3.72 4.78a.75.75 0 0 1 0-1.06Z"></path>

</svg></button>

</header>

<input-demux data-action="tab-container-change:input-demux#storeInput tab-container-changed:input-demux#updateInput">

<tab-container class="d-flex flex-column js-branches-tags-tabs" style="min-height: 0;">

<div class="SelectMenu-filter">

<input data-target="input-demux.source"

id="context-commitish-filter-field"

class="SelectMenu-input form-control"

aria-owns="ref-list-branches"

data-controls-ref-menu-id="ref-list-branches"

autofocus

autocomplete="off"

aria-label="Filter branches/tags"

placeholder="Filter branches/tags"

type="text"

>

</div>

<div class="SelectMenu-tabs" role="tablist" data-target="input-demux.control" >

<button class="SelectMenu-tab" type="button" role="tab" aria-selected="true">Branches</button>

<button class="SelectMenu-tab" type="button" role="tab">Tags</button>

</div>

<div role="tabpanel" id="ref-list-branches" data-filter-placeholder="Filter branches/tags" tabindex="" class="d-flex flex-column flex-auto overflow-auto">

<ref-selector

type="branch"

data-targets="input-demux.sinks"

data-action="

input-entered:ref-selector#inputEntered

tab-selected:ref-selector#tabSelected

focus-list:ref-selector#focusFirstListMember

"

query-endpoint="/KEERRO/ctf-writeups/refs"



cache-key="v0:1647876588.2277062"

current-committish="bWFzdGVy"

default-branch="bWFzdGVy"

name-with-owner="S0VFUlJPL2N0Zi13cml0ZXVwcw=="

prefetch-on-mouseover

>

<template data-target="ref-selector.fetchFailedTemplate">

<div class="SelectMenu-message" data-index="{{ index }}">Could not load branches</div>

</template>

<template data-target="ref-selector.noMatchTemplate">

<div class="SelectMenu-message">Nothing to show</div>

</template>

<div data-target="ref-selector.listContainer" role="menu" class="SelectMenu-list " data-turbo-frame="repo-content-turbo-frame">

<div class="SelectMenu-loading pt-3 pb-0 overflow-hidden" aria-label="Menu is loading">

<svg style="box-sizing: content-box; color: var(--color-icon-primary);" width="32" height="32" viewBox="0 0 16 16" fill="none" data-view-component="true" class="anim-rotate">

<circle cx="8" cy="8" r="7" stroke="currentColor" stroke-opacity="0.25" stroke-width="2" vector-effect="non-scaling-stroke" />

<path d="M15 8a7.002 7.002 0 00-7-7" stroke="currentColor" stroke-width="2" stroke-linecap="round" vector-effect="non-scaling-stroke" />

</svg>

</div>

</div>

<template data-target="ref-selector.itemTemplate">



<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-check SelectMenu-icon SelectMenu-icon--check">

<path d="M13.78 4.22a.75.75 0 0 1 0 1.06l-7.25 7.25a.75.75 0 0 1-1.06 0L2.22 9.28a.751.751 0 0 1 .018-1.042.751.751 0 0 1 1.042-.018L6 10.94l6.72-6.72a.75.75 0 0 1 1.06 0Z"></path>

</svg>

<span>{{ refName }}</span>

<span>default</span>



</template>

<footer class="SelectMenu-footer">View all branches</footer>

</ref-selector>

</div>

<div role="tabpanel" id="tags-menu" data-filter-placeholder="Find a tag" tabindex="" hidden class="d-flex flex-column flex-auto overflow-auto">

<ref-selector

type="tag"

data-action="

input-entered:ref-selector#inputEntered

tab-selected:ref-selector#tabSelected

focus-list:ref-selector#focusFirstListMember

"

data-targets="input-demux.sinks"

query-endpoint="/KEERRO/ctf-writeups/refs"

cache-key="v0:1647876588.2277062"

current-committish="bWFzdGVy"

default-branch="bWFzdGVy"

name-with-owner="S0VFUlJPL2N0Zi13cml0ZXVwcw=="

>

<template data-target="ref-selector.fetchFailedTemplate">

<div class="SelectMenu-message" data-index="{{ index }}">Could not load tags</div>

</template>

<template data-target="ref-selector.noMatchTemplate">

<div class="SelectMenu-message" data-index="{{ index }}">Nothing to show</div>

</template>

<template data-target="ref-selector.itemTemplate">



<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-check SelectMenu-icon SelectMenu-icon--check">

<path d="M13.78 4.22a.75.75 0 0 1 0 1.06l-7.25 7.25a.75.75 0 0 1-1.06 0L2.22 9.28a.751.751 0 0 1 .018-1.042.751.751 0 0 1 1.042-.018L6 10.94l6.72-6.72a.75.75 0 0 1 1.06 0Z"></path>

</svg>

<span>{{ refName }}</span>

<span>default</span>



</template>

<div data-target="ref-selector.listContainer" role="menu" class="SelectMenu-list" data-turbo-frame="repo-content-turbo-frame">

<div class="SelectMenu-loading pt-3 pb-0 overflow-hidden" aria-label="Menu is loading">

<svg style="box-sizing: content-box; color: var(--color-icon-primary);" width="32" height="32" viewBox="0 0 16 16" fill="none" data-view-component="true" class="anim-rotate">

<circle cx="8" cy="8" r="7" stroke="currentColor" stroke-opacity="0.25" stroke-width="2" vector-effect="non-scaling-stroke" />

<path d="M15 8a7.002 7.002 0 00-7-7" stroke="currentColor" stroke-width="2" stroke-linecap="round" vector-effect="non-scaling-stroke" />

</svg>

</div>

</div>

<footer class="SelectMenu-footer">View all tags</footer>

</ref-selector>

</div>

</tab-container>

</input-demux>

</div>

</div>

</details>

</div>

<div class="Overlay--hidden Overlay-backdrop--center" data-modal-dialog-overlay>

<modal-dialog role="dialog" id="warn-tag-match-create-branch-dialog" aria-modal="true" aria-labelledby="warn-tag-match-create-branch-dialog-header" data-view-component="true" class="Overlay Overlay--width-large Overlay--height-auto Overlay--motion-scaleFade">

<header class="Overlay-header Overlay-header--large Overlay-header--divided">

<div class="Overlay-headerContentWrap">

<div class="Overlay-titleWrap">

<h1 id="warn-tag-match-create-branch-dialog-header" class="Overlay-title">Name already in use</h1>

</div>

<div class="Overlay-actionWrap">

<button data-close-dialog-id="warn-tag-match-create-branch-dialog" aria-label="Close" type="button" data-view-component="true" class="close-button Overlay-closeButton"><svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-x">

<path d="M3.72 3.72a.75.75 0 0 1 1.06 0L8 6.94l3.22-3.22a.749.749 0 0 1 1.275.326.749.749 0 0 1-.215.734L9.06 8l3.22 3.22a.749.749 0 0 1-.326 1.275.749.749 0 0 1-.734-.215L8 9.06l-3.22 3.22a.751.751 0 0 1-1.042-.018.751.751 0 0 1-.018-1.042L6.94 8 3.72 4.78a.75.75 0 0 1 0-1.06Z"></path>

</svg></button>

</div>

</div>

</header>

<div class="Overlay-body ">



<div data-view-component="true"> A tag already exists with the provided branch name. Many Git commands accept both tag and branch names, so creating this branch may cause unexpected behavior. Are you sure you want to create this branch?

</div>

</div>

<footer class="Overlay-footer Overlay-footer--alignEnd">

<button data-close-dialog-id="warn-tag-match-create-branch-dialog" type="button" data-view-component="true" class="btn"> Cancel

</button>

<button data-submit-dialog-id="warn-tag-match-create-branch-dialog" type="button" data-view-component="true" class="btn-danger btn"> Create

</button>

</footer>

</modal-dialog></div>

<div class="flex-1 mx-2 flex-self-center f4">

<div class="d-none d-sm-block">

<span><span><span>ctf-writeups</span></span></span><span>/</span><span><span>Hack Zone Tunisia 2020</span></span><span>/</span><span><span>pwn</span></span><span>/</span>pwn2<span>/</span>

</div>

</div>

<div class="d-flex">



Go to file



</div>

</div>

<div class="f4 mt-3 mb-3 d-sm-none"><span><span><span>ctf-writeups</span></span></span><span>/</span><span><span>Hack Zone Tunisia 2020</span></span><span>/</span><span><span>pwn</span></span><span>/</span>pwn2<span>/</span></div>

<div class="Box mb-3" >

<div class="Box-header position-relative">

<h2 class="sr-only">Latest commit</h2>

<div class="js-details-container Details d-flex rounded-top-2 flex-items-center flex-wrap" data-issue-and-pr-hovercards-enabled>

<include-fragment src="/KEERRO/ctf-writeups/tree-commit/06bbaff46db8a3bdd138b18de938f9440e4e83b8/Hack%20Zone%20Tunisia%202020/pwn/pwn2" class="d-flex flex-auto flex-items-center" aria-busy="true" aria-label="Loading latest commit">

<div class="Skeleton avatar avatar-user flex-shrink-0 ml-n1 mr-n1 mt-n1 mb-n1" style="width:24px;height:24px;"></div>

<div class="Skeleton Skeleton--text col-5 ml-3"> </div>

</include-fragment> <div class="flex-shrink-0">

<h2 class="sr-only">Git stats</h2>



Files

<include-fragment src="/KEERRO/ctf-writeups/file-list/master/Hack%20Zone%20Tunisia%202020/pwn/pwn2">

Permalink

<div data-view-component="true" class="include-fragment-error flash flash-error flash-full py-2">

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-alert">

<path d="M6.457 1.047c.659-1.234 2.427-1.234 3.086 0l6.082 11.378A1.75 1.75 0 0 1 14.082 15H1.918a1.75 1.75 0 0 1-1.543-2.575Zm1.763.707a.25.25 0 0 0-.44 0L1.698 13.132a.25.25 0 0 0 .22.368h12.164a.25.25 0 0 0 .22-.368Zm.53 3.996v2.5a.75.75 0 0 1-1.5 0v-2.5a.75.75 0 0 1 1.5 0ZM9 11a1 1 0 1 1-2 0 1 1 0 0 1 2 0Z"></path>

</svg>

Failed to load latest commit information.



</div> <div class="js-details-container Details" data-hpc>

<div role="grid" aria-labelledby="files" class="Details-content--hidden-not-important js-navigation-container js-active-navigation-container d-block">

<div class="sr-only" role="row">

<div role="columnheader">Type</div>

<div role="columnheader">Name</div>

<div role="columnheader" class="d-none d-md-block">Latest commit message</div>

<div role="columnheader">Commit time</div>

</div>

<div role="row" class="Box-row Box-row--focus-gray p-0 d-flex js-navigation-item" >

<div role="rowheader" class="flex-auto min-width-0 col-md-2">



<span>. .</span>

</div>

<div role="gridcell" class="d-none d-md-block"></div>

<div role="gridcell"></div>

</div>

<div role="row" class="Box-row Box-row--focus-gray py-2 d-flex position-relative js-navigation-item ">

<div role="gridcell" class="mr-3 flex-shrink-0" style="width: 16px;">

<svg aria-label="File" aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-file color-fg-muted">

<path d="M2 1.75C2 .784 2.784 0 3.75 0h6.586c.464 0 .909.184 1.237.513l2.914 2.914c.329.328.513.773.513 1.237v9.586A1.75 1.75 0 0 1 13.25 16h-9.5A1.75 1.75 0 0 1 2 14.25Zm1.75-.25a.25.25 0 0 0-.25.25v12.5c0 .138.112.25.25.25h9.5a.25.25 0 0 0 .25-.25V6h-2.75A1.75 1.75 0 0 1 9 4.25V1.5Zm6.75.062V4.25c0 .138.112.25.25.25h2.688l-.011-.013-2.914-2.914-.013-.011Z"></path>

</svg>

</div>

<div role="rowheader" class="flex-auto min-width-0 col-md-2 mr-3">

<span>sploit.py</span>

</div>

<div role="gridcell" class="flex-auto min-width-0 d-none d-md-block col-5 mr-3" >

<div class="Skeleton Skeleton--text col-7"> </div>

</div>

<div role="gridcell" class="color-fg-muted text-right" style="width:100px;">

<div class="Skeleton Skeleton--text"> </div>

</div>

</div>

</div>

</div>

</include-fragment>

</div>

</div>

</div>

</div>

</turbo-frame>
