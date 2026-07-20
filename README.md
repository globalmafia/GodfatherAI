# GodfatherAI
Home based affiliate Ai trading bot

# GodfatherAI

## Setup

1. Clone repo
2. `npm install`
3. Add `.env.local` with all keys (Supabase + Stripe)
4. Deploy Edge Function: `supabase functions deploy validate-payment`
5. Add Stripe webhook pointing to `/api/webhooks/stripe`
6. Run `npm run dev`


import { headers } from 'next/headers'
import Stripe from 'stripe'
import { createClient } from '@supabase/supabase-js'
import { NextResponse } from 'next/server'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)
const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!)

export async function POST(req: Request) {
  const body = await req.text()
  const sig = headers().get('stripe-signature')!

  const event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!)

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object as Stripe.Checkout.Session
    await supabase.from('profiles').update({ subscription_status: 'active' }).eq('id', session.metadata?.supabase_user_id)
  }

  return NextResponse.json({ received: true })
}

import { createClient } from '@supabase/supabase-js'
import Stripe from 'stripe'
import { NextResponse } from 'next/server'

const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!)
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-06-20' })

export async function POST(req: Request) {
  const { name, email, telegram, membership, coin } = await req.json()

  const { data: user } = await supabase.auth.admin.createUser({
    email,
    password: 'TempPass123!ChangeMe',
    email_confirm: true,
  })

  const customer = await stripe.customers.create({
    email,
    name,
    metadata: { supabase_id: user.user.id }
  })

  const session = await stripe.checkout.sessions.create({
    customer: customer.id,
    mode: membership === 'monthly' ? 'subscription' : 'payment',
    line_items: [{
      price: membership === 'monthly' ? process.env.STRIPE_MONTHLY_PRICE_ID : process.env.STRIPE_ANNUAL_PRICE_ID,
      quantity: 1,
    }],
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/dashboard`,
    metadata: { supabase_user_id: user.user.id, coin }
  })

  await supabase.from('profiles').insert({
    id: user.user.id,
    full_name: name,
    telegram,
    membership_type: membership,
    stripe_customer_id: customer.id,
    subscription_status: 'pending'
  })

  return NextResponse.json({ checkout_url: session.url })
}

'use client'

import { useState } from 'react'

export default function GodfatherAI() {
  const [step, setStep] = useState<'landing' | 'initiate' | 'payment'>('landing')
  const [refId, setRefId] = useState('')
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    telegram: '',
    membership: 'monthly',
    coin: 'USDT_TRC20'
  })
  const [txid, setTxid] = useState('')
  const [loading, setLoading] = useState(false)

  const wallets: Record<string, string> = {
    USDT_TRC20: "TRC20_USDT_YOUR_REAL_ADDRESS_HERE",
    USD_CASHAPP: "$GlobalRyzeMafia"
  }

  const handleInitiate = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)

    const res = await fetch('/api/initiate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(formData)
    })

    const data = await res.json()
    if (data.checkout_url) {
      window.location.href = data.checkout_url
    } else {
      alert("Error starting checkout")
    }
    setLoading(false)
  }

  return (
    <div className="min-h-screen bg-[#0a0a0a] text-white">
      <nav className="border-b border-[#3a2f1f] bg-black/90 px-8 py-5 flex justify-between">
        <div className="flex items-center gap-x-3">
          <div className="w-10 h-10 bg-[#c5a46e] rounded-2xl flex items-center justify-center">
            <span className="text-black font-bold text-2xl">G</span>
          </div>
          <span className="text-3xl font-bold tracking-tighter text-[#c5a46e]">GODFATHER</span>
          <span className="text-white/70">AI</span>
        </div>
        <button onClick={() => setStep('initiate')} className="bg-[#c5a46e] text-black px-6 py-2.5 rounded-2xl font-semibold">
          Join the Family
        </button>
      </nav>

      {step === 'landing' && (
        <div className="max-w-5xl mx-auto px-6 pt-20 text-center">
          <div className="text-[120px] mb-6">👑</div>
          <h1 className="text-7xl font-bold tracking-tighter">THE GOD FATHER</h1>
          <p className="mt-4 text-2xl text-[#c5a46e]">Global Ryze Mafia</p>
          <button onClick={() => setStep('initiate')} className="mt-10 bg-[#c5a46e] text-black px-10 py-4 rounded-2xl text-xl font-bold">
            Begin Initiation
          </button>
        </div>
      )}

      {step === 'initiate' && (
        <div className="max-w-xl mx-auto px-6 py-16">
          <div className="bg-[#111] border border-[#c5a46e] rounded-3xl p-10">
            <h2 className="text-4xl font-bold text-[#c5a46e] mb-8">Global Mafia Initiation</h2>
            <form onSubmit={handleInitiate} className="space-y-6">
              <input type="text" placeholder="Full Name" className="w-full p-4 bg-[#0a0a0a] border border-[#c5a46e] rounded-2xl" required onChange={e => setFormData({...formData, name: e.target.value})} />
              <input type="email" placeholder="Email" className="w-full p-4 bg-[#0a0a0a] border border-[#c5a46e] rounded-2xl" required onChange={e => setFormData({...formData, email: e.target.value})} />
              <input type="text" placeholder="Telegram @username" className="w-full p-4 bg-[#0a0a0a] border border-[#c5a46e] rounded-2xl" required onChange={e => setFormData({...formData, telegram: e.target.value})} />

              <select className="w-full p-4 bg-[#0a0a0a] border border-[#c5a46e] rounded-2xl" onChange={e => setFormData({...formData, membership: e.target.value})}>
                <option value="monthly">Monthly — $97/month</option>
                <option value="annual">Annual — $797/year</option>
              </select>

              <select className="w-full p-4 bg-[#0a0a0a] border border-[#c5a46e] rounded-2xl" onChange={e => setFormData({...formData, coin: e.target.value})}>
                <option value="USDT_TRC20">USDT (TRC20) — Recommended</option>
                <option value="USD_CASHAPP">USD via Cash App</option>
              </select>

              <button type="submit" className="w-full py-5 bg-[#c5a46e] text-black rounded-2xl font-bold text-xl">CONTINUE TO PAYMENT</button>
            </form>
          </div>
        </div>
      )}
    </div>
  )
}


{
  "name": "godfather-ai",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "14.2.3",
    "react": "^18",
    "react-dom": "^18",
    "@supabase/supabase-js": "^2.43.4",
    "stripe": "^15.10.0"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^18",
    "typescript": "^5"
  }
}

godfatherAI/
├── app/
│   ├── api/
│   │   ├── initiate/route.ts
│   │   └── webhooks/stripe/route.ts
│   ├── dashboard/page.tsx
│   ├── layout.tsx
│   ├── page.tsx
│   └── globals.css
├── components/
├── lib/
│   └── supabase.ts
├── supabase/
│   └── functions/
│       └── validate-payment/
├── .env.example
├── package.json
├── next.config.js
└── README.md
