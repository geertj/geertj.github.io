---
layout: post
title: "Tax Reporting for Ethereum mining"
comments: true
tags: blockchain
---

It is tax season again. By April 18, millions of Americans need to submit their
tax returns. And as a permanent resident in my new country, I need to do the
same. The difference is that this year it will be even more fun, as I did some
Ethereum mining.

<!--more-->

My mining activities have created income for me, and so it is no surprise that
the [IRS](https://www.irs.gov/uac/newsroom/irs-virtual-currency-guidance) wants
to collect taxes on it. [Notice
2014-21](https://www.irs.gov/pub/irs-drop/n-14-21.pdf) contains a lot of detail
on how the IRS looks at virtual currencies. In the case of mining, it explains
that this needs to be reported as income. But how do you determine the income
you generated?

There's a great site at [bitcoin.tax](https://bitcoin.tax) that can help you
determine your mining income. The site allows you to put in your mining
addresses, after which it will download the associated mining transactions from
the blockchain, match them with the price of Ethereum at the transaction time,
and add everything up.  The result is a report with your total mining income.
This process also gives you a cost basis for your coins, so that if you ever
sell them again in the future (a separate taxable event), you can calculate
your gains.

But, what fun is just using the bitcoin.tax site if we can write a small script
that does the same thing? So I wrote a Python script that does exactly that,
which you can [download
here](https://gist.github.com/geertj/81dbb52e10821101a9f5d9ba774ff90d).

The script requires the [requests](http://docs.python-requests.org/) and
[pytz](https://pypi.python.org/pypi/pytz) modules, so install those if you
don't have them. You may want to set up a [virtual
environment](https://docs.python.org/3/library/venv.html), and you also need to
make sure that you do that with Python3.

{% highlight shell-session %}
$ python3 -mvenv eth
$ . eth/bin/activate
$ pip install requests pytz
{% endhighlight %}

Use the script as follows:

{% highlight shell-session %}
$ python txlog.py
Usage: txlog.py <address> [<year>]
{% endhighlight %}

Running the script on my mining account (obfuscating some details for obvious
reasons):

{% highlight shell-session %}
$ python txlog.py my.account.hash 2016
Downloading transactions from etherscan.io...
=> Status: 200
=> Downloaded XXX transactions.
Downloading historical prices from etherchain.org...
=> Status: 200
=> Downloaded 13,801 historical prices.
Pricing transactions...
=> Average price delay 28m, max price delay 57m.
Filtering transactions for year 2016...
=> Filtered YYY transactions
Writing CSV file...
=> Wrote YYY transactions to transactions.csv.
{% endhighlight %}

Some features of the script:

* It downloads transaction data using the
  [etherscan.io](https://etherscan.io/apis) public API. The API is nice and you
  only need one call to get all transactions for a single account.

* It downloads bulk pricing data once, from
  [etherchain.org](https://etherchain.org/api/statistics/price), and then uses
  that to price all transactions. Earlier versions were using a different API
  to price a single transaction, but this turned out to be very slow.

* The script tells you the average and maximum "staleness" of the pricing data
  that is used.  The etherchain pricing usually has a granularity of one hour.

* It saves the transaction and pricing data to a JSON file. This allows you to
  reconstruct the mining revenue you calculated at any time in the future
  (maybe you get audited).

The output of the run above is a CSV file with all transactions, that can be
imported into a spreadsheet for further processing. I imported it into [Google
Sheets](https://docs.google.com/spreadsheets), and added two things:

* A **type** column, where I tag mining transactions as "mining". In my case
  mining transactions are easily identified as they are all for little over 1
  ether, and come from just 3 different pool addresses.

* A call to [SUMIF](https://support.google.com/docs/answer/3093583?hl=en)
  to calculate the total revenue from all the tagged mining transactions.

As a last step I compared my results to those I got from bitcoin.tax. The
difference was less than 3 dollars, likely caused by using slightly different
pricing data. So I'm pretty confident that the results are correct.

That's it for this post folks. Hope it was useful!
