INSTALLATION
============

To install, just add the app to your settings.py INSTALLED_APPS like:

```python
INSTALLED_APPS = [
    ...
    'django_bitcoin',
    ...
]
```

Also you have to run a local bitcoind instance, and specify connection string in settings.

```python
BITCOIND_CONNECTION_STRING = "http://bitcoinuser:password@localhost:8332"
```

USAGE
=====

### Tutorial

There is a small tutorial about how to use django-bitcoin to create your own instawallet: http://blog.kangasbros.fi/?p=85

### Wallet websites, escrow services using the "Wallet"-model

You can use the `Wallet` class to do different bitcoin-moving applications. Typical example would be a marketplace-style site, where there are multiple sellers and buyer. Or job freelance site, where escrow is needed. Or even an exchange could be done with this abstraction (a little extra classes would be needed however).

Note that while you move bitcoins between Wallet-objects, only bitcoin transactions needed are incoming and outgoing transactions. 
Transactions between the system "Wallet"-objects don't generate "real" bitcoin transactions. Every transaction (except incoming transactions) is logged to `WalletTransaction` object to ease accounting.

This also means that outgoing bitcoin transactions are "mixed".

```python
from django_bitcoin import Wallet, currency

class Profile(models.Model):
    wallet = ForeignKey(Wallet)
    outgoing_bitcoin_address = CharField()

class Escrow(models.Model):
    wallet = ForeignKey(Wallet)
    buyer_happy = BooleanField(default=False)

buyer=Profile.objects.create()
seller=Profile.objects.create()

purchase=Escrow.objects.create()

AMOUNT_USD="9.99"

m=currency.Money(AMOUNT_USD, "USD")
btc_amount=currency.exchange(m, "BTC")

print "Send "+str(btc_amount)+" BTC to address "+buyer.wallet.receiving_address()

sleep(5000) # wait for transaction

if p1.wallet.total_balance()>=btc_amount:
    p1.send_to_wallet(purchase, btc_amount)

    sleep(1000) # wait for product/service delivery

    if purchase.buyer_happy:
        purchase.wallet.send_to_wallet(seller.wallet)
        seller.wallet.send_to_address(seller.outgoing_bitcoin_address, seller.wallet.total_balance())
    else:
        print "WHY U NO HAPPY"
        #return bitcoins to buyer, 50/50 split or something
```

### Templatetags

To display transaction history and simple wallet tagline in your views, use the following templatetags:

```django
{% load currency_conversions %}
<!-- display balance tagline, estimate in USD and received/sent -->
{% wallet_tagline profile.bitcoin_wallet %}
<!-- display list of transactions as a table -->
{% wallet_history profile.bitcoin_wallet %} 
```

Easy way to convert currencies from each other: `btc2usd, usd2btc, eur2btc, btc2eur`

Also currency2btc, btc2currency for any currencies on bitcoincharts.com

```django
{% load currency_conversions %}
Hi, for the pizza: send me {{bitcoin_amount}}BTC (about {{ bitcoin_amount|btc2usd }}USD).
```

Display QR code of the bitcoin payment using google charts API.

```python
{% load currency_conversions %}
Pay the following payment with your android bitcoin wallet:
{% bitcoin_payment_qr wallet.receiving_address bitcoin_amount %}.

The same but display also description and an estimate in EUR:
{% bitcoin_payment_qr wallet.receiving_address bitcoin_amount "One beer" "EUR" %}.
```

### Transaction notifications

To enable bitcoin transaction notifications, set the following flag in your settings.py

BITCOIN_TRANSACTION_SIGNALING = True

After that, you need to setup a cron job to run each minute, something like the following:

```cron
* * * * * (cd $APP_PATH && python manage.py python manage.py CheckTransactions >> $APP_PATH/logs/email_sends.log 2>&1)
```

After that you can define your balance_changed and balance_changed_confirmed signals:

```python
from django_bitcoin.models import balance_changed, balance_changed_confirmed
from django.dispatch import receiver


@receiver(balance_changed)
def balance_changed_handler(sender, **kwargs):
    pass
    # try:
    # print "balance changed", sender.id, kwargs["changed"], sender.total_balance()


@receiver(balance_changed_confirmed)
def balance_changed_confirmed_handler(sender, **kwargs):
    pass
```

### obsolete

There is older Payment -class, which can be used for simpler things (direct payments etc):

```python
from django_bitcoin import Payment, new_bitcoin_payment, bitcoinprice
from decimal import Decimal

bp=new_bitcoin_payment(Decimal("0.32"))
bp2=new_bitcoin_payment(Decimal("0.99")/bitcoinprice("USD")) # convert from USD

if bp.is_paid(minconf=5):
    # send 5% of payment to bitcoin address someaddress
    bp.withdraw_proportion(someaddress, Decimal("5.0"))
    if bp.is_paid() and bp2.is_paid():
        # send from both bp and bp2 (95% from both, only single bitcoin transaction)
        Payment.withdraw_all(someaddress2,  {bp, bp2})
```

USED AT
=======

Currently django-bitcoin is used at production in bountychest.com

More to come!

If you have a site using django-bitcoin, drop me an email and I will link to it here.

NOTE
====

I don't have time to answer to email support requests, sorry guys.


