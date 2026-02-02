# Race Results Multi-Tenant App - Implementation Plan

## Overview

You're building a modern web app where:
- Users sign up and pick a username (e.g., `jake`)
- They get a public page at `jake.raceresults.site`
- They can add/edit their race results via a dashboard
- No manual JSON editing required

---

## Quick Glossary (PHP/MySQL → Modern Equivalents)

| Old World | New World | What It Means |
|-----------|-----------|---------------|
| Shared hosting (cPanel) | **Vercel** | Hosts your app, handles deployments |
| FTP upload | **Git push** | You push code, it auto-deploys |
| PHP | **Next.js (JavaScript/React)** | Server + frontend in one |
| MySQL | **PostgreSQL (via Supabase)** | Same concept, different database |
| phpMyAdmin | **Supabase Dashboard** | Visual database management |
| Hand-coded login system | **Supabase Auth** | Handles signup/login/sessions for you |
| `.htaccess` rewrites | **Middleware** | Code that runs before each request |
| `$_SESSION` | **JWT tokens/cookies** | Handled automatically by auth library |

---

## Services You'll Use

### 1. GitHub (Free)
**What:** Stores your code, tracks changes
**Why:** Vercel deploys directly from GitHub - push code, site updates automatically
**Account:** https://github.com (you may already have this)

### 2. Vercel (Free tier)
**What:** Hosts your Next.js app
**Why:** Best-in-class Next.js hosting, handles wildcard subdomains, auto-SSL
**Account:** https://vercel.com (sign up with GitHub)

### 3. Supabase (Free tier)
**What:** PostgreSQL database + authentication + API
**Why:** Gives you a database and user auth without building it yourself
**Account:** https://supabase.com

### 4. Cloudflare (Free tier)
**What:** DNS management
**Why:** Easy wildcard DNS setup, also adds caching/security
**Account:** https://cloudflare.com

### 5. Domain Registrar
**What:** Where you buy `raceresults.site`
**Options:** Namecheap, Porkbun, Cloudflare Registrar, Google Domains
**Cost:** ~$10-15/year

---

## Project Structure

You'll have **one repository** with this structure:

```
raceresults-app/
├── app/                      # Next.js app directory
│   ├── page.tsx              # Landing page (raceresults.site)
│   ├── layout.tsx            # Shared layout wrapper
│   ├── login/
│   │   └── page.tsx          # Login page
│   ├── signup/
│   │   └── page.tsx          # Signup page
│   ├── dashboard/
│   │   ├── page.tsx          # User's private dashboard
│   │   ├── races/
│   │   │   ├── page.tsx      # List/manage races
│   │   │   ├── new/
│   │   │   │   └── page.tsx  # Add new race form
│   │   │   └── [id]/
│   │   │       └── edit/
│   │   │           └── page.tsx  # Edit race form
│   │   └── settings/
│   │       └── page.tsx      # Account settings
│   └── user/
│       └── [username]/
│           └── page.tsx      # Public profile (rewritten from subdomain)
├── components/               # Reusable UI components
│   ├── RaceTable.tsx
│   ├── PersonalBests.tsx
│   ├── UpcomingRaces.tsx
│   ├── Statistics.tsx
│   └── ...
├── lib/
│   ├── supabase.ts          # Database client setup
│   └── utils.ts             # Helper functions
├── middleware.ts            # Subdomain routing logic
├── package.json             # Dependencies
└── README.md
```

---

## Phase 0: Prerequisites & Account Setup

### Time estimate: 1-2 hours

### Step 0.1: Install Required Software

You'll need these installed on your computer:

**Node.js** (JavaScript runtime - like PHP but for JS)
- Download from: https://nodejs.org
- Choose the LTS (Long Term Support) version
- Verify installation: open terminal, run `node --version`

**Git** (version control)
- You may already have this
- Verify: run `git --version`
- If not installed: https://git-scm.com/downloads

**A code editor**
- Recommended: VS Code (https://code.visualstudio.com)
- Has great support for JavaScript/TypeScript

### Step 0.2: Create Accounts

Create accounts on these services (all have generous free tiers):

1. **GitHub** - https://github.com/signup
   - This is where your code lives
   - Vercel will read from here to deploy

2. **Vercel** - https://vercel.com/signup
   - Click "Continue with GitHub" to link accounts
   - This is where your app runs

3. **Supabase** - https://supabase.com
   - Click "Start your project"
   - Sign up with GitHub for convenience
   - This is your database and auth

4. **Cloudflare** - https://cloudflare.com
   - Sign up for free account
   - This manages your DNS

### Step 0.3: Buy Your Domain

Purchase `raceresults.site` from any registrar:
- **Porkbun** - Often cheapest, good UI
- **Namecheap** - Popular, reliable
- **Cloudflare Registrar** - At-cost pricing, already integrated

Don't configure anything yet - we'll set up DNS later.

---

## Phase 1: Set Up the Database (Supabase)

### Time estimate: 30 minutes

### Step 1.1: Create a Supabase Project

1. Go to https://supabase.com/dashboard
2. Click "New Project"
3. Fill in:
   - **Name:** `raceresults`
   - **Database Password:** Generate a strong one, save it somewhere safe
   - **Region:** Choose closest to you/your users
4. Click "Create new project"
5. Wait 2-3 minutes for it to provision

### Step 1.2: Create Database Tables

1. In your Supabase project, go to **SQL Editor** (left sidebar)
2. Click "New query"
3. Paste this SQL and click "Run":

```sql
-- Create profiles table (extends Supabase auth.users)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE NOT NULL,
  display_name TEXT,
  bio TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create races table
CREATE TABLE races (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  date DATE NOT NULL,
  distance_km DECIMAL(6,2),
  distance_mi DECIMAL(6,2),
  time_seconds INTEGER,  -- Store as seconds, format on display
  pace_km TEXT,          -- "5:30" per km
  pace_mi TEXT,          -- "8:51" per mi
  race_website TEXT,
  results_website TEXT,
  strava_activity TEXT,
  personal_best BOOLEAN DEFAULT FALSE,
  notes TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create goals table
CREATE TABLE goals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  target TEXT,
  status TEXT DEFAULT 'in_progress' CHECK (status IN ('completed', 'in_progress', 'upcoming')),
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create indexes for faster queries
CREATE INDEX races_user_id_idx ON races(user_id);
CREATE INDEX races_date_idx ON races(date DESC);
CREATE INDEX goals_user_id_idx ON goals(user_id);
CREATE INDEX profiles_username_idx ON profiles(username);

-- Enable Row Level Security (RLS) - IMPORTANT for multi-tenant
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE races ENABLE ROW LEVEL SECURITY;
ALTER TABLE goals ENABLE ROW LEVEL SECURITY;

-- Profiles: Anyone can read, users can only update their own
CREATE POLICY "Public profiles are viewable by everyone"
  ON profiles FOR SELECT
  USING (true);

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id);

CREATE POLICY "Users can insert own profile"
  ON profiles FOR INSERT
  WITH CHECK (auth.uid() = id);

-- Races: Anyone can read, users can only modify their own
CREATE POLICY "Races are viewable by everyone"
  ON races FOR SELECT
  USING (true);

CREATE POLICY "Users can insert own races"
  ON races FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own races"
  ON races FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own races"
  ON races FOR DELETE
  USING (auth.uid() = user_id);

-- Goals: Anyone can read, users can only modify their own
CREATE POLICY "Goals are viewable by everyone"
  ON goals FOR SELECT
  USING (true);

CREATE POLICY "Users can insert own goals"
  ON goals FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own goals"
  ON goals FOR UPDATE
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own goals"
  ON goals FOR DELETE
  USING (auth.uid() = user_id);
```

### Step 1.3: Set Up Auth Trigger

When a user signs up, we need to automatically create their profile. Run this SQL:

```sql
-- Function to create profile on signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, username, display_name)
  VALUES (
    NEW.id,
    NEW.raw_user_meta_data->>'username',
    NEW.raw_user_meta_data->>'display_name'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger to call function on signup
CREATE OR REPLACE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

### Step 1.4: Configure Auth Settings

1. In Supabase, go to **Authentication** → **Providers**
2. Make sure **Email** is enabled
3. Go to **Authentication** → **URL Configuration**
4. Note down:
   - Site URL (we'll update this later)
   - Redirect URLs (we'll add these later)

### Step 1.5: Get Your API Keys

1. Go to **Settings** → **API** (left sidebar)
2. Note down these values (you'll need them later):
   - **Project URL** - looks like `https://xxxxx.supabase.co`
   - **anon/public key** - long string starting with `eyJ...`
   - Keep the `service_role` key secret - don't put it in frontend code

---

## Phase 2: Create the Next.js App

### Time estimate: 1-2 hours

### Step 2.1: Create New Project

Open your terminal and run:

```bash
# Create a new Next.js app
npx create-next-app@latest raceresults-app

# When prompted, choose:
# ✔ Would you like to use TypeScript? Yes
# ✔ Would you like to use ESLint? Yes
# ✔ Would you like to use Tailwind CSS? Yes
# ✔ Would you like to use `src/` directory? No
# ✔ Would you like to use App Router? Yes
# ✔ Would you like to customize the default import alias? No

# Go into the project
cd raceresults-app

# Install Supabase libraries
npm install @supabase/supabase-js @supabase/ssr
```

### Step 2.2: Set Up Environment Variables

Create a file called `.env.local` in your project root:

```bash
# Supabase (get these from Step 1.5)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key-here

# Your domain (for subdomain detection)
NEXT_PUBLIC_ROOT_DOMAIN=raceresults.site
```

**Important:** The `.env.local` file is gitignored by default - it won't be uploaded to GitHub. You'll add these same values to Vercel later.

### Step 2.3: Create Supabase Client

Create `lib/supabase.ts`:

```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

Create `lib/supabase-server.ts`:

```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createServerSupabaseClient() {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Ignore - called from Server Component
          }
        },
      },
    }
  )
}
```

### Step 2.4: Create Subdomain Middleware

Create `middleware.ts` in the project root:

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const host = request.headers.get('host') || ''
  const rootDomain = process.env.NEXT_PUBLIC_ROOT_DOMAIN || 'raceresults.site'

  // Get subdomain
  // host could be: "jake.raceresults.site" or "localhost:3000"
  let subdomain: string | null = null

  if (host.includes(rootDomain)) {
    const parts = host.split('.')
    if (parts.length > 2 || (parts.length === 2 && !parts[0].includes('localhost'))) {
      subdomain = parts[0]
    }
  }

  // Skip for main domain, www, and special routes
  if (!subdomain || subdomain === 'www' || subdomain === 'app') {
    return NextResponse.next()
  }

  // Rewrite to user route
  const url = request.nextUrl.clone()
  url.pathname = `/user/${subdomain}${url.pathname === '/' ? '' : url.pathname}`

  return NextResponse.rewrite(url)
}

export const config = {
  matcher: [
    // Match all paths except static files and api routes
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

### Step 2.5: Create Basic Pages

I'll outline the key files you need to create. Each file goes in the `app/` directory.

**app/page.tsx** - Landing page:

```tsx
import Link from 'next/link'

export default function HomePage() {
  return (
    <main className="min-h-screen bg-gradient-to-b from-gray-900 to-gray-800 text-white">
      <div className="container mx-auto px-4 py-20">
        <h1 className="text-5xl font-bold text-center mb-6">
          Race Results
        </h1>
        <p className="text-xl text-center text-gray-300 mb-12">
          Track your running journey. Share your achievements.
        </p>

        <div className="flex justify-center gap-4">
          <Link
            href="/signup"
            className="bg-blue-600 hover:bg-blue-700 px-6 py-3 rounded-lg font-semibold"
          >
            Get Started Free
          </Link>
          <Link
            href="/login"
            className="border border-gray-600 hover:border-gray-500 px-6 py-3 rounded-lg"
          >
            Log In
          </Link>
        </div>
      </div>
    </main>
  )
}
```

**app/user/[username]/page.tsx** - Public profile page:

```tsx
import { createServerSupabaseClient } from '@/lib/supabase-server'
import { notFound } from 'next/navigation'
import RaceTable from '@/components/RaceTable'
import PersonalBests from '@/components/PersonalBests'

interface Props {
  params: { username: string }
}

export default async function UserProfilePage({ params }: Props) {
  const { username } = params
  const supabase = await createServerSupabaseClient()

  // Fetch user profile
  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('username', username)
    .single()

  if (!profile) {
    notFound()
  }

  // Fetch user's races
  const { data: races } = await supabase
    .from('races')
    .select('*')
    .eq('user_id', profile.id)
    .order('date', { ascending: false })

  return (
    <main className="min-h-screen bg-gray-50">
      <header className="bg-white border-b">
        <div className="container mx-auto px-4 py-6">
          <h1 className="text-3xl font-bold">
            {profile.display_name || profile.username}
          </h1>
          {profile.bio && (
            <p className="text-gray-600 mt-2">{profile.bio}</p>
          )}
        </div>
      </header>

      <div className="container mx-auto px-4 py-8">
        <PersonalBests races={races || []} />
        <RaceTable races={races || []} />
      </div>
    </main>
  )
}
```

### Step 2.6: Port Your Existing UI Components

You'll want to convert your existing HTML/CSS/JS into React components. The structure will be:

```
components/
├── RaceTable.tsx        # The sortable race history table
├── PersonalBests.tsx    # The PB cards grid
├── UpcomingRaces.tsx    # Future races with countdown
├── Statistics.tsx       # The year/distance matrix
├── Goals.tsx            # Goals list
├── UnitToggle.tsx       # Metric/Imperial toggle
└── RaceForm.tsx         # Form for adding/editing races
```

This is where most of the work is - converting your vanilla JS to React. The logic stays the same, but the structure changes. For example:

**Before (vanilla JS):**
```javascript
function renderRaceTable(races) {
  const html = races.map(race => `
    <tr>
      <td>${race.name}</td>
      <td>${race.date}</td>
    </tr>
  `).join('')
  document.getElementById('race-table').innerHTML = html
}
```

**After (React):**
```tsx
function RaceTable({ races }) {
  return (
    <table>
      <tbody>
        {races.map(race => (
          <tr key={race.id}>
            <td>{race.name}</td>
            <td>{race.date}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

---

## Phase 3: Build Authentication

### Time estimate: 2-3 hours

### Step 3.1: Create Auth Pages

**app/signup/page.tsx:**

```tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase'
import Link from 'next/link'

export default function SignUpPage() {
  const router = useRouter()
  const supabase = createClient()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setLoading(true)
    setError(null)

    const formData = new FormData(e.currentTarget)
    const email = formData.get('email') as string
    const password = formData.get('password') as string
    const username = formData.get('username') as string
    const displayName = formData.get('displayName') as string

    // Check if username is available
    const { data: existing } = await supabase
      .from('profiles')
      .select('username')
      .eq('username', username.toLowerCase())
      .single()

    if (existing) {
      setError('Username is already taken')
      setLoading(false)
      return
    }

    // Sign up
    const { error: signUpError } = await supabase.auth.signUp({
      email,
      password,
      options: {
        data: {
          username: username.toLowerCase(),
          display_name: displayName,
        },
      },
    })

    if (signUpError) {
      setError(signUpError.message)
      setLoading(false)
      return
    }

    // Redirect to dashboard
    router.push('/dashboard')
  }

  return (
    <main className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-lg shadow-md w-full max-w-md">
        <h1 className="text-2xl font-bold mb-6">Create your account</h1>

        {error && (
          <div className="bg-red-100 text-red-700 p-3 rounded mb-4">
            {error}
          </div>
        )}

        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label className="block text-sm font-medium mb-1">
              Username
            </label>
            <div className="flex items-center">
              <input
                name="username"
                type="text"
                required
                pattern="[a-zA-Z0-9_-]+"
                className="flex-1 border rounded-l px-3 py-2"
                placeholder="jake"
              />
              <span className="bg-gray-100 border border-l-0 rounded-r px-3 py-2 text-gray-500">
                .raceresults.site
              </span>
            </div>
          </div>

          <div className="mb-4">
            <label className="block text-sm font-medium mb-1">
              Display Name
            </label>
            <input
              name="displayName"
              type="text"
              className="w-full border rounded px-3 py-2"
              placeholder="Jake Walker"
            />
          </div>

          <div className="mb-4">
            <label className="block text-sm font-medium mb-1">
              Email
            </label>
            <input
              name="email"
              type="email"
              required
              className="w-full border rounded px-3 py-2"
            />
          </div>

          <div className="mb-6">
            <label className="block text-sm font-medium mb-1">
              Password
            </label>
            <input
              name="password"
              type="password"
              required
              minLength={8}
              className="w-full border rounded px-3 py-2"
            />
          </div>

          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 rounded font-semibold hover:bg-blue-700 disabled:opacity-50"
          >
            {loading ? 'Creating account...' : 'Create Account'}
          </button>
        </form>

        <p className="mt-4 text-center text-gray-600">
          Already have an account?{' '}
          <Link href="/login" className="text-blue-600 hover:underline">
            Log in
          </Link>
        </p>
      </div>
    </main>
  )
}
```

**app/login/page.tsx:**

```tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase'
import Link from 'next/link'

export default function LoginPage() {
  const router = useRouter()
  const supabase = createClient()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setLoading(true)
    setError(null)

    const formData = new FormData(e.currentTarget)
    const email = formData.get('email') as string
    const password = formData.get('password') as string

    const { error: signInError } = await supabase.auth.signInWithPassword({
      email,
      password,
    })

    if (signInError) {
      setError(signInError.message)
      setLoading(false)
      return
    }

    router.push('/dashboard')
  }

  return (
    <main className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-lg shadow-md w-full max-w-md">
        <h1 className="text-2xl font-bold mb-6">Log in</h1>

        {error && (
          <div className="bg-red-100 text-red-700 p-3 rounded mb-4">
            {error}
          </div>
        )}

        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label className="block text-sm font-medium mb-1">Email</label>
            <input
              name="email"
              type="email"
              required
              className="w-full border rounded px-3 py-2"
            />
          </div>

          <div className="mb-6">
            <label className="block text-sm font-medium mb-1">Password</label>
            <input
              name="password"
              type="password"
              required
              className="w-full border rounded px-3 py-2"
            />
          </div>

          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 rounded font-semibold hover:bg-blue-700 disabled:opacity-50"
          >
            {loading ? 'Logging in...' : 'Log In'}
          </button>
        </form>

        <p className="mt-4 text-center text-gray-600">
          Don't have an account?{' '}
          <Link href="/signup" className="text-blue-600 hover:underline">
            Sign up
          </Link>
        </p>
      </div>
    </main>
  )
}
```

---

## Phase 4: Build the Dashboard

### Time estimate: 4-6 hours

This is where users manage their races. Key pages:

### Step 4.1: Dashboard Layout

**app/dashboard/layout.tsx:**

```tsx
import { createServerSupabaseClient } from '@/lib/supabase-server'
import { redirect } from 'next/navigation'
import Link from 'next/link'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const supabase = await createServerSupabaseClient()

  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    redirect('/login')
  }

  const { data: profile } = await supabase
    .from('profiles')
    .select('username, display_name')
    .eq('id', user.id)
    .single()

  return (
    <div className="min-h-screen bg-gray-100">
      <nav className="bg-white shadow">
        <div className="container mx-auto px-4 py-4 flex items-center justify-between">
          <div className="flex items-center gap-6">
            <Link href="/dashboard" className="font-bold text-xl">
              Race Results
            </Link>
            <Link href="/dashboard/races" className="text-gray-600 hover:text-gray-900">
              My Races
            </Link>
            <Link href="/dashboard/goals" className="text-gray-600 hover:text-gray-900">
              Goals
            </Link>
            <Link href="/dashboard/settings" className="text-gray-600 hover:text-gray-900">
              Settings
            </Link>
          </div>
          <div className="flex items-center gap-4">
            <a
              href={`https://${profile?.username}.raceresults.site`}
              target="_blank"
              className="text-blue-600 hover:underline"
            >
              View my page →
            </a>
            <form action="/api/auth/signout" method="POST">
              <button className="text-gray-600 hover:text-gray-900">
                Log out
              </button>
            </form>
          </div>
        </div>
      </nav>

      <main className="container mx-auto px-4 py-8">
        {children}
      </main>
    </div>
  )
}
```

### Step 4.2: Races List Page

**app/dashboard/races/page.tsx:**

```tsx
import { createServerSupabaseClient } from '@/lib/supabase-server'
import Link from 'next/link'
import DeleteRaceButton from '@/components/DeleteRaceButton'

export default async function RacesPage() {
  const supabase = await createServerSupabaseClient()

  const { data: { user } } = await supabase.auth.getUser()

  const { data: races } = await supabase
    .from('races')
    .select('*')
    .eq('user_id', user!.id)
    .order('date', { ascending: false })

  return (
    <div>
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold">My Races</h1>
        <Link
          href="/dashboard/races/new"
          className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
        >
          + Add Race
        </Link>
      </div>

      <div className="bg-white rounded-lg shadow overflow-hidden">
        <table className="w-full">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-4 py-3 text-left">Date</th>
              <th className="px-4 py-3 text-left">Race</th>
              <th className="px-4 py-3 text-left">Distance</th>
              <th className="px-4 py-3 text-left">Time</th>
              <th className="px-4 py-3 text-left">PB</th>
              <th className="px-4 py-3 text-left">Actions</th>
            </tr>
          </thead>
          <tbody>
            {races?.map(race => (
              <tr key={race.id} className="border-t">
                <td className="px-4 py-3">{race.date}</td>
                <td className="px-4 py-3 font-medium">{race.name}</td>
                <td className="px-4 py-3">{race.distance_km} km</td>
                <td className="px-4 py-3 font-mono">
                  {formatTime(race.time_seconds)}
                </td>
                <td className="px-4 py-3">
                  {race.personal_best && '🏆'}
                </td>
                <td className="px-4 py-3">
                  <Link
                    href={`/dashboard/races/${race.id}/edit`}
                    className="text-blue-600 hover:underline mr-3"
                  >
                    Edit
                  </Link>
                  <DeleteRaceButton raceId={race.id} />
                </td>
              </tr>
            ))}
          </tbody>
        </table>

        {(!races || races.length === 0) && (
          <div className="p-8 text-center text-gray-500">
            No races yet. Add your first race to get started!
          </div>
        )}
      </div>
    </div>
  )
}

function formatTime(seconds: number): string {
  if (!seconds) return '-'
  const h = Math.floor(seconds / 3600)
  const m = Math.floor((seconds % 3600) / 60)
  const s = seconds % 60
  if (h > 0) {
    return `${h}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`
  }
  return `${m}:${s.toString().padStart(2, '0')}`
}
```

### Step 4.3: Add/Edit Race Form

**app/dashboard/races/new/page.tsx:**

```tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase'

export default function NewRacePage() {
  const router = useRouter()
  const supabase = createClient()
  const [loading, setLoading] = useState(false)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setLoading(true)

    const formData = new FormData(e.currentTarget)

    const { data: { user } } = await supabase.auth.getUser()

    // Parse time string to seconds
    const timeStr = formData.get('time') as string
    const timeSeconds = parseTimeToSeconds(timeStr)

    const distanceKm = parseFloat(formData.get('distanceKm') as string)
    const distanceMi = distanceKm * 0.621371

    // Calculate pace
    const paceKm = formatPace(timeSeconds / distanceKm)
    const paceMi = formatPace(timeSeconds / distanceMi)

    const { error } = await supabase.from('races').insert({
      user_id: user!.id,
      name: formData.get('name'),
      date: formData.get('date'),
      distance_km: distanceKm,
      distance_mi: distanceMi.toFixed(2),
      time_seconds: timeSeconds,
      pace_km: paceKm,
      pace_mi: paceMi,
      race_website: formData.get('raceWebsite') || null,
      results_website: formData.get('resultsWebsite') || null,
      strava_activity: formData.get('stravaActivity') || null,
      personal_best: formData.get('personalBest') === 'on',
    })

    if (error) {
      console.error('Error adding race:', error)
      setLoading(false)
      return
    }

    router.push('/dashboard/races')
  }

  return (
    <div className="max-w-2xl">
      <h1 className="text-2xl font-bold mb-6">Add New Race</h1>

      <form onSubmit={handleSubmit} className="bg-white rounded-lg shadow p-6">
        <div className="grid grid-cols-2 gap-4 mb-4">
          <div className="col-span-2">
            <label className="block text-sm font-medium mb-1">Race Name *</label>
            <input
              name="name"
              type="text"
              required
              className="w-full border rounded px-3 py-2"
              placeholder="London Marathon 2024"
            />
          </div>

          <div>
            <label className="block text-sm font-medium mb-1">Date *</label>
            <input
              name="date"
              type="date"
              required
              className="w-full border rounded px-3 py-2"
            />
          </div>

          <div>
            <label className="block text-sm font-medium mb-1">Distance (km) *</label>
            <input
              name="distanceKm"
              type="number"
              step="0.01"
              required
              className="w-full border rounded px-3 py-2"
              placeholder="42.195"
            />
          </div>

          <div>
            <label className="block text-sm font-medium mb-1">Finish Time *</label>
            <input
              name="time"
              type="text"
              required
              pattern="(\d+:)?\d{1,2}:\d{2}"
              className="w-full border rounded px-3 py-2"
              placeholder="3:45:22 or 25:30"
            />
          </div>

          <div className="flex items-center">
            <input
              name="personalBest"
              type="checkbox"
              id="personalBest"
              className="mr-2"
            />
            <label htmlFor="personalBest">Personal Best</label>
          </div>
        </div>

        <div className="border-t pt-4 mt-4">
          <h3 className="font-medium mb-3">Optional Links</h3>
          <div className="grid grid-cols-1 gap-4">
            <div>
              <label className="block text-sm font-medium mb-1">Race Website</label>
              <input
                name="raceWebsite"
                type="url"
                className="w-full border rounded px-3 py-2"
                placeholder="https://..."
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">Results Link</label>
              <input
                name="resultsWebsite"
                type="url"
                className="w-full border rounded px-3 py-2"
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">Strava Activity</label>
              <input
                name="stravaActivity"
                type="url"
                className="w-full border rounded px-3 py-2"
              />
            </div>
          </div>
        </div>

        <div className="flex gap-3 mt-6">
          <button
            type="submit"
            disabled={loading}
            className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 disabled:opacity-50"
          >
            {loading ? 'Saving...' : 'Save Race'}
          </button>
          <button
            type="button"
            onClick={() => router.back()}
            className="border px-4 py-2 rounded hover:bg-gray-50"
          >
            Cancel
          </button>
        </div>
      </form>
    </div>
  )
}

function parseTimeToSeconds(time: string): number {
  const parts = time.split(':').map(Number)
  if (parts.length === 3) {
    return parts[0] * 3600 + parts[1] * 60 + parts[2]
  }
  return parts[0] * 60 + parts[1]
}

function formatPace(secondsPerUnit: number): string {
  const mins = Math.floor(secondsPerUnit / 60)
  const secs = Math.round(secondsPerUnit % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}
```

---

## Phase 5: Deploy to Vercel

### Time estimate: 30 minutes - 1 hour

### Step 5.1: Push to GitHub

```bash
# In your project directory
git init
git add .
git commit -m "Initial commit"

# Create repo on GitHub, then:
git remote add origin https://github.com/YOUR_USERNAME/raceresults-app.git
git branch -M main
git push -u origin main
```

### Step 5.2: Connect to Vercel

1. Go to https://vercel.com/dashboard
2. Click "Add New..." → "Project"
3. Select your `raceresults-app` repository
4. Configure:
   - **Framework Preset:** Next.js (auto-detected)
   - **Environment Variables:** Add these:
     - `NEXT_PUBLIC_SUPABASE_URL` = your Supabase URL
     - `NEXT_PUBLIC_SUPABASE_ANON_KEY` = your anon key
     - `NEXT_PUBLIC_ROOT_DOMAIN` = raceresults.site
5. Click "Deploy"
6. Wait for deployment to complete

### Step 5.3: Configure Custom Domain

1. In Vercel project settings, go to "Domains"
2. Add `raceresults.site`
3. Add `*.raceresults.site` (wildcard)
4. Vercel will show you DNS records to add

### Step 5.4: Configure Cloudflare DNS

1. In your domain registrar, change nameservers to Cloudflare's
2. In Cloudflare dashboard, add your domain
3. Add DNS records:

```
Type    Name    Content              Proxy
A       @       76.76.21.21          Yes
CNAME   *       cname.vercel-dns.com Yes
CNAME   www     cname.vercel-dns.com Yes
```

(Vercel will give you the exact values)

4. Wait for DNS propagation (can take up to 24-48 hours, usually faster)

### Step 5.5: Update Supabase URLs

1. Go to Supabase → Authentication → URL Configuration
2. Update:
   - **Site URL:** `https://raceresults.site`
   - **Redirect URLs:** Add:
     - `https://raceresults.site/**`
     - `https://*.raceresults.site/**`

---

## Phase 6: Data Migration

### Time estimate: 30 minutes

Once everything is working, import your existing race data.

### Step 6.1: Create Import Script

Create a one-time script or use the Supabase SQL editor:

```sql
-- Example: Import your existing data
-- You'll need to first create your user account, then get your user ID

INSERT INTO races (user_id, name, date, distance_km, distance_mi, time_seconds, pace_km, pace_mi, personal_best)
VALUES
  ('YOUR-USER-UUID', 'Tewkesbury Half Marathon', '2024-10-06', 21.1, 13.11, 6144, '4:51', '7:49', true),
  ('YOUR-USER-UUID', 'London Marathon', '2024-04-21', 42.195, 26.22, 14722, '5:49', '9:21', false)
  -- ... more races
;
```

### Step 6.2: Build a JSON Importer

Or build a one-time import feature in the dashboard that reads your existing `races.json` format and inserts into the database.

---

## Summary Checklist

### Phase 0: Setup (1-2 hours)
- [ ] Install Node.js
- [ ] Install Git
- [ ] Install VS Code
- [ ] Create GitHub account
- [ ] Create Vercel account
- [ ] Create Supabase account
- [ ] Create Cloudflare account
- [ ] Buy domain (raceresults.site)

### Phase 1: Database (30 mins)
- [ ] Create Supabase project
- [ ] Run SQL to create tables
- [ ] Set up auth trigger
- [ ] Note down API keys

### Phase 2: Next.js App (1-2 hours)
- [ ] Create Next.js project
- [ ] Install Supabase libraries
- [ ] Set up environment variables
- [ ] Create Supabase client files
- [ ] Create middleware for subdomains
- [ ] Create basic pages

### Phase 3: Authentication (2-3 hours)
- [ ] Create signup page
- [ ] Create login page
- [ ] Test user creation

### Phase 4: Dashboard (4-6 hours)
- [ ] Create dashboard layout
- [ ] Create races list page
- [ ] Create add race form
- [ ] Create edit race form
- [ ] Port existing UI components

### Phase 5: Deploy (30 mins - 1 hour)
- [ ] Push to GitHub
- [ ] Deploy on Vercel
- [ ] Configure custom domain
- [ ] Set up Cloudflare DNS
- [ ] Update Supabase URLs

### Phase 6: Migration (30 mins)
- [ ] Import existing race data
- [ ] Verify everything works

---

## Cost Summary

| Item | Monthly Cost |
|------|--------------|
| Domain (raceresults.site) | ~$1/mo ($10-15/year) |
| Vercel (free tier) | $0 |
| Supabase (free tier) | $0 |
| Cloudflare (free tier) | $0 |
| **Total** | **~$1/month** |

Free tiers are generous enough for hundreds of users. You'd only need to upgrade if the app becomes very popular.

---

## Future Enhancements

Once the basics work, you could add:

- **Strava Integration:** Auto-import races from Strava
- **Custom Themes:** Let users customize colors
- **Custom Domains:** Allow `mydomain.com` → `user.raceresults.site`
- **Social Features:** Follow other runners, compare PRs
- **Race Calendar:** Browse upcoming races
- **Training Log:** Track regular runs, not just races
- **Embeddable Widgets:** "Add to your blog" code snippets

---

## Getting Help

When you get stuck:

1. **Next.js Docs:** https://nextjs.org/docs
2. **Supabase Docs:** https://supabase.com/docs
3. **Vercel Docs:** https://vercel.com/docs
4. **Stack Overflow:** Search "[next.js] your issue"
5. **Supabase Discord:** Active community for help
