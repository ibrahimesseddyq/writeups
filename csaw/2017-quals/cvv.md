# CVV

**CTF:** CSAW 2017 Quals  
**Category:** pwn  
**Original writeup:** <https://github.com/Lyusternik/CTFWriteups/tree/master/CSAW%20-17/CVV> — by **Lyusternik**. All credit goes to the original author; this is an archived copy.

---

<span><span><span>CTFWriteups</span></span></span><span>/</span><span><span>CSAW -17</span></span><span>/</span>CVV<span>/</span>

</div>

</div>

<div class="d-flex">



Go to file



</div>

</div>

<div class="f4 mt-3 mb-3 d-sm-none"><span><span><span>CTFWriteups</span></span></span><span>/</span><span><span>CSAW -17</span></span><span>/</span>CVV<span>/</span></div>

<div class="Box mb-3" >

<div class="Box-header position-relative">

<h2 class="sr-only">Latest commit</h2>

<div class="js-details-container Details d-flex rounded-top-2 flex-items-center flex-wrap" data-issue-and-pr-hovercards-enabled>

<include-fragment src="/Lyusternik/CTFWriteups/tree-commit/f406abec8acb59bbd9c71a836493971b88b09bf4/CSAW%20-17/CVV" class="d-flex flex-auto flex-items-center" aria-busy="true" aria-label="Loading latest commit">

<div class="Skeleton avatar avatar-user flex-shrink-0 ml-n1 mr-n1 mt-n1 mb-n1" style="width:24px;height:24px;"></div>

<div class="Skeleton Skeleton--text col-5 ml-3"> </div>

</include-fragment> <div class="flex-shrink-0">

<h2 class="sr-only">Git stats</h2>



Files

<include-fragment src="/Lyusternik/CTFWriteups/file-list/master/CSAW%20-17/CVV">

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

<span>CVV explanation.txt</span>

</div>

<div role="gridcell" class="flex-auto min-width-0 d-none d-md-block col-5 mr-3" >

<div class="Skeleton Skeleton--text col-7"> </div>

</div>

<div role="gridcell" class="color-fg-muted text-right" style="width:100px;">

<div class="Skeleton Skeleton--text"> </div>

</div>

</div>

<div role="row" class="Box-row Box-row--focus-gray py-2 d-flex position-relative js-navigation-item ">

<div role="gridcell" class="mr-3 flex-shrink-0" style="width: 16px;">

<svg aria-label="File" aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-file color-fg-muted">

<path d="M2 1.75C2 .784 2.784 0 3.75 0h6.586c.464 0 .909.184 1.237.513l2.914 2.914c.329.328.513.773.513 1.237v9.586A1.75 1.75 0 0 1 13.25 16h-9.5A1.75 1.75 0 0 1 2 14.25Zm1.75-.25a.25.25 0 0 0-.25.25v12.5c0 .138.112.25.25.25h9.5a.25.25 0 0 0 .25-.25V6h-2.75A1.75 1.75 0 0 1 9 4.25V1.5Zm6.75.062V4.25c0 .138.112.25.25.25h2.688l-.011-.013-2.914-2.914-.013-.011Z"></path>

</svg>

</div>

<div role="rowheader" class="flex-auto min-width-0 col-md-2 mr-3">

<span>cvv.py</span>

</div>

<div role="gridcell" class="flex-auto min-width-0 d-none d-md-block col-5 mr-3" >
