# AfroHubMarket — Production-Ready MVP

---

## 1. PROJECT STRUCTURE

```
afrohubmarket/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── admin/page.tsx
│   ├── api/
│   │   ├── auth/[...nextauth]/route.ts
│   │   ├── businesses/route.ts
│   │   ├── businesses/[id]/route.ts
│   │   ├── stripe/route.ts
│   │   ├── stripe/webhook/route.ts
│   │   └── upload/route.ts
│   ├── blog/
│   │   ├── [slug]/page.tsx
│   │   └── page.tsx
│   ├── business/[id]/page.tsx
│   ├── dashboard/
│   │   ├── create/page.tsx
│   │   └── page.tsx
│   ├── directory/page.tsx
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── BusinessCard.tsx
│   ├── BusinessForm.tsx
│   ├── DashboardLayout.tsx
│   ├── Footer.tsx
│   ├── Navbar.tsx
│   └── SearchFilters.tsx
├── lib/
│   ├── auth.ts
│   ├── prisma.ts
│   ├── stripe.ts
│   └── validations.ts
├── prisma/schema.prisma
├── types/index.ts
├── .env.example
├── next.config.ts
├── tailwind.config.ts
└── tsconfig.json
```

---

## 2. package.json

```json
{
  "name": "afrohubmarket",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:studio": "prisma studio",
    "db:seed": "tsx prisma/seed.ts"
  },
  "dependencies": {
    "@next-auth/prisma-adapter": "^1.0.7",
    "@prisma/client": "^5.10.0",
    "@stripe/stripe-js": "^3.0.0",
    "bcryptjs": "^2.4.3",
    "cloudinary": "^2.0.0",
    "next": "14.1.0",
    "next-auth": "^4.24.5",
    "react": "^18",
    "react-dom": "^18",
    "stripe": "^14.17.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.6",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "autoprefixer": "^10.0.1",
    "eslint": "^8",
    "eslint-config-next": "14.1.0",
    "postcss": "^8",
    "prisma": "^5.10.0",
    "tailwindcss": "^3.3.0",
    "tsx": "^4.7.0",
    "typescript": "^5"
  }
}
```

---

## 3. .env.example

```env
# App
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-here-generate-with-openssl-rand-base64-32

# Database (Neon or Supabase PostgreSQL)
DATABASE_URL=postgresql://user:password@host/dbname?sslmode=require

# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_FEATURED_PRICE_ID=price_...

# Cloudinary
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# Next public
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 4. prisma/schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  USER
  BUSINESS_OWNER
  ADMIN
}

enum ListingStatus {
  PENDING
  APPROVED
  REJECTED
}

enum SubscriptionStatus {
  ACTIVE
  CANCELED
  PAST_DUE
  INACTIVE
}

model User {
  id             String    @id @default(cuid())
  name           String?
  email          String    @unique
  emailVerified  DateTime?
  password       String?
  image          String?
  role           Role      @default(USER)
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt

  accounts       Account[]
  sessions       Session[]
  businesses     Business[]
  subscription   Subscription?
}

model Account {
  id                 String  @id @default(cuid())
  userId             String
  type               String
  provider           String
  providerAccountId  String
  refresh_token      String? @db.Text
  access_token       String? @db.Text
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String? @db.Text
  session_state      String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

model Category {
  id         String     @id @default(cuid())
  name       String     @unique
  slug       String     @unique
  icon       String?
  businesses Business[]
}

model Business {
  id          String        @id @default(cuid())
  name        String
  slug        String        @unique
  description String        @db.Text
  address     String
  city        String        @default("Berlin")
  phone       String?
  whatsapp    String?
  website     String?
  images      String[]
  featured    Boolean       @default(false)
  status      ListingStatus @default(PENDING)
  viewCount   Int           @default(0)
  whatsappClicks Int        @default(0)
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])

  ownerId     String
  owner       User     @relation(fields: [ownerId], references: [id], onDelete: Cascade)
}

model Subscription {
  id                   String             @id @default(cuid())
  userId               String             @unique
  stripeCustomerId     String?            @unique
  stripeSubscriptionId String?            @unique
  stripePriceId        String?
  status               SubscriptionStatus @default(INACTIVE)
  currentPeriodEnd     DateTime?
  createdAt            DateTime           @default(now())
  updatedAt            DateTime           @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Post {
  id          String   @id @default(cuid())
  title       String
  slug        String   @unique
  excerpt     String
  content     String   @db.Text
  coverImage  String?
  published   Boolean  @default(false)
  tags        String[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

---

## 5. lib/prisma.ts

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({ log: ["query"] });

if (process.env.NODE_ENV !== "production")
  globalForPrisma.prisma = prisma;
```

---

## 6. lib/auth.ts

```typescript
import { NextAuthOptions } from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import CredentialsProvider from "next-auth/providers/credentials";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import bcrypt from "bcryptjs";
import { prisma } from "./prisma";

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;

        const user = await prisma.user.findUnique({
          where: { email: credentials.email },
        });

        if (!user || !user.password) return null;

        const isValid = await bcrypt.compare(
          credentials.password,
          user.password
        );
        if (!isValid) return null;

        return user;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
        token.id = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.role = token.role as string;
        session.user.id = token.id as string;
      }
      return session;
    },
  },
  session: { strategy: "jwt" },
  pages: {
    signIn: "/login",
    error: "/login",
  },
  secret: process.env.NEXTAUTH_SECRET,
};
```

---

## 7. lib/stripe.ts

```typescript
import Stripe from "stripe";

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2023-10-16",
  typescript: true,
});

export const FEATURED_PRICE_ID = process.env.STRIPE_FEATURED_PRICE_ID!;

export async function getOrCreateStripeCustomer(
  userId: string,
  email: string,
  name?: string | null
): Promise<string> {
  const { prisma } = await import("./prisma");
  let sub = await prisma.subscription.findUnique({ where: { userId } });

  if (sub?.stripeCustomerId) return sub.stripeCustomerId;

  const customer = await stripe.customers.create({ email, name: name ?? undefined });

  await prisma.subscription.upsert({
    where: { userId },
    update: { stripeCustomerId: customer.id },
    create: { userId, stripeCustomerId: customer.id },
  });

  return customer.id;
}
```

---

## 8. lib/validations.ts

```typescript
import { z } from "zod";

export const registerSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});

export const businessSchema = z.object({
  name: z.string().min(2, "Business name is required"),
  description: z.string().min(20, "Description must be at least 20 characters"),
  categoryId: z.string().min(1, "Category is required"),
  address: z.string().min(5, "Address is required"),
  city: z.string().default("Berlin"),
  phone: z.string().optional(),
  whatsapp: z.string().optional(),
  website: z.string().url().optional().or(z.literal("")),
  images: z.array(z.string()).optional().default([]),
});

export const postSchema = z.object({
  title: z.string().min(5),
  slug: z.string().min(3),
  excerpt: z.string().min(20),
  content: z.string().min(50),
  coverImage: z.string().optional(),
  published: z.boolean().default(false),
  tags: z.array(z.string()).default([]),
});

export type RegisterInput = z.infer<typeof registerSchema>;
export type LoginInput = z.infer<typeof loginSchema>;
export type BusinessInput = z.infer<typeof businessSchema>;
```

---

## 9. types/index.ts

```typescript
import { Business, Category, User, Subscription, Post } from "@prisma/client";

export type BusinessWithCategory = Business & {
  category: Category;
  owner: Pick<User, "id" | "name" | "email">;
};

export type BusinessWithAll = BusinessWithCategory & {
  owner: User;
};

export type UserWithSubscription = User & {
  subscription: Subscription | null;
};

declare module "next-auth" {
  interface Session {
    user: {
      id: string;
      name?: string | null;
      email?: string | null;
      image?: string | null;
      role: string;
    };
  }
}
```

---

## 10. next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "res.cloudinary.com" },
      { protocol: "https", hostname: "lh3.googleusercontent.com" },
    ],
  },
};

export default nextConfig;
```

---

## 11. tailwind.config.ts

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#fef7ee",
          100: "#fdedd8",
          500: "#f97316",
          600: "#ea6c0a",
          700: "#c2530a",
          900: "#7c2d12",
        },
        afro: {
          green: "#16a34a",
          gold: "#d97706",
          red: "#dc2626",
        },
      },
      fontFamily: {
        sans: ["Inter", "sans-serif"],
      },
    },
  },
  plugins: [],
};

export default config;
```

---

## 12. app/globals.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  body {
    @apply bg-white text-gray-900 font-sans;
  }
}

@layer components {
  .btn-primary {
    @apply bg-brand-500 hover:bg-brand-600 text-white font-semibold px-6 py-2.5 rounded-lg transition-colors duration-200;
  }
  .btn-secondary {
    @apply border border-gray-300 hover:bg-gray-50 text-gray-700 font-semibold px-6 py-2.5 rounded-lg transition-colors duration-200;
  }
  .input {
    @apply w-full border border-gray-300 rounded-lg px-4 py-2.5 focus:outline-none focus:ring-2 focus:ring-brand-500 focus:border-transparent;
  }
  .card {
    @apply bg-white rounded-xl shadow-sm border border-gray-100 overflow-hidden hover:shadow-md transition-shadow duration-200;
  }
  .badge {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium;
  }
}
```

---

## 13. app/layout.tsx

```typescript
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { Providers } from "./providers";
import Navbar from "@/components/Navbar";
import Footer from "@/components/Footer";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: {
    default: "AfroHubMarket — African Businesses in Berlin & Germany",
    template: "%s | AfroHubMarket",
  },
  description:
    "Discover and connect with African-owned businesses, services, and opportunities across Berlin and Germany. Your trusted African diaspora marketplace.",
  keywords: [
    "African businesses Berlin",
    "African diaspora Germany",
    "African market Berlin",
    "African restaurant Berlin",
    "AfroHubMarket",
  ],
  openGraph: {
    type: "website",
    locale: "en_DE",
    url: process.env.NEXT_PUBLIC_APP_URL,
    siteName: "AfroHubMarket",
  },
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <Providers>
          <div className="min-h-screen flex flex-col">
            <Navbar />
            <main className="flex-1">{children}</main>
            <Footer />
          </div>
        </Providers>
      </body>
    </html>
  );
}
```

---

## 14. app/providers.tsx

```typescript
"use client";

import { SessionProvider } from "next-auth/react";

export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

---

## 15. app/page.tsx (Homepage)

```typescript
import type { Metadata } from "next";
import Link from "next/link";
import Image from "next/image";
import { prisma } from "@/lib/prisma";
import BusinessCard from "@/components/BusinessCard";
import SearchFilters from "@/components/SearchFilters";

export const metadata: Metadata = {
  title: "AfroHubMarket — African Businesses in Berlin",
  description:
    "Find African restaurants, salons, shops and services in Berlin. Your gateway to the African community in Germany.",
};

const CATEGORIES = [
  { name: "Restaurant", slug: "restaurant", icon: "🍽️" },
  { name: "Hair & Beauty", slug: "hair-beauty", icon: "💇" },
  { name: "Grocery & Shop", slug: "grocery", icon: "🛒" },
  { name: "Logistics", slug: "logistics", icon: "📦" },
  { name: "Finance", slug: "finance", icon: "💰" },
  { name: "Legal & Advice", slug: "legal", icon: "⚖️" },
  { name: "Events", slug: "events", icon: "🎉" },
  { name: "Health", slug: "health", icon: "🏥" },
];

async function getFeaturedBusinesses() {
  return prisma.business.findMany({
    where: { featured: true, status: "APPROVED" },
    include: { category: true, owner: { select: { id: true, name: true, email: true } } },
    take: 6,
    orderBy: { createdAt: "desc" },
  });
}

async function getRecentBusinesses() {
  return prisma.business.findMany({
    where: { status: "APPROVED" },
    include: { category: true, owner: { select: { id: true, name: true, email: true } } },
    take: 8,
    orderBy: { createdAt: "desc" },
  });
}

export default async function HomePage() {
  const [featured, recent] = await Promise.all([
    getFeaturedBusinesses(),
    getRecentBusinesses(),
  ]);

  return (
    <div>
      {/* HERO */}
      <section className="bg-gradient-to-br from-brand-700 via-brand-600 to-afro-gold text-white">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-20 md:py-28">
          <div className="text-center max-w-3xl mx-auto">
            <span className="inline-block bg-white/20 text-white text-sm font-medium px-4 py-1.5 rounded-full mb-6">
              🌍 The African Diaspora Marketplace in Germany
            </span>
            <h1 className="text-4xl md:text-6xl font-bold mb-6 leading-tight">
              Discover African Businesses in Berlin
            </h1>
            <p className="text-xl text-white/90 mb-10">
              Connect with African-owned restaurants, salons, shops, and
              services. Support your community, grow your network.
            </p>
            <div className="flex flex-col sm:flex-row gap-4 justify-center">
              <Link href="/directory" className="btn-primary bg-white text-brand-700 hover:bg-gray-100 text-lg px-8 py-3">
                Browse Directory
              </Link>
              <Link href="/dashboard/create" className="btn-secondary border-white text-white hover:bg-white/10 text-lg px-8 py-3">
                List Your Business
              </Link>
            </div>
          </div>
        </div>
      </section>

      {/* SEARCH BAR */}
      <section className="bg-white shadow-sm border-b">
        <div className="max-w-4xl mx-auto px-4 py-6">
          <SearchFilters compact />
        </div>
      </section>

      {/* CATEGORIES */}
      <section className="py-16 bg-gray-50">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <h2 className="text-2xl font-bold text-gray-900 mb-8 text-center">
            Browse by Category
          </h2>
          <div className="grid grid-cols-2 sm:grid-cols-4 lg:grid-cols-8 gap-4">
            {CATEGORIES.map((cat) => (
              <Link
                key={cat.slug}
                href={`/directory?category=${cat.slug}`}
                className="flex flex-col items-center p-4 bg-white rounded-xl border border-gray-100 hover:border-brand-500 hover:shadow-md transition-all group"
              >
                <span className="text-3xl mb-2">{cat.icon}</span>
                <span className="text-sm font-medium text-gray-700 group-hover:text-brand-600 text-center">
                  {cat.name}
                </span>
              </Link>
            ))}
          </div>
        </div>
      </section>

      {/* FEATURED LISTINGS */}
      {featured.length > 0 && (
        <section className="py-16">
          <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div className="flex justify-between items-center mb-8">
              <div>
                <span className="text-brand-500 font-semibold text-sm uppercase tracking-wide">
                  ⭐ Featured
                </span>
                <h2 className="text-2xl font-bold text-gray-900 mt-1">
                  Top Businesses This Month
                </h2>
              </div>
              <Link href="/directory?featured=true" className="text-brand-500 hover:text-brand-600 font-medium">
                View all →
              </Link>
            </div>
            <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
              {featured.map((biz) => (
                <BusinessCard key={biz.id} business={biz} />
              ))}
            </div>
          </div>
        </section>
      )}

      {/* RECENT LISTINGS */}
      <section className="py-16 bg-gray-50">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between items-center mb-8">
            <h2 className="text-2xl font-bold text-gray-900">
              Recently Added
            </h2>
            <Link href="/directory" className="text-brand-500 hover:text-brand-600 font-medium">
              Browse all →
            </Link>
          </div>
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-6">
            {recent.map((biz) => (
              <BusinessCard key={biz.id} business={biz} compact />
            ))}
          </div>
        </div>
      </section>

      {/* CTA BANNER */}
      <section className="py-20 bg-brand-700 text-white">
        <div className="max-w-4xl mx-auto px-4 text-center">
          <h2 className="text-3xl font-bold mb-4">
            Own a Business? Get Listed Today
          </h2>
          <p className="text-white/80 text-lg mb-8">
            Reach thousands of Africans in Berlin. Free basic listing —
            upgrade to featured for €20/month.
          </p>
          <Link href="/dashboard/create" className="btn-primary bg-white text-brand-700 hover:bg-gray-100 text-lg px-10 py-3">
            Add Your Business Free
          </Link>
        </div>
      </section>
    </div>
  );
}
```

---

## 16. app/directory/page.tsx

```typescript
import type { Metadata } from "next";
import { prisma } from "@/lib/prisma";
import BusinessCard from "@/components/BusinessCard";
import SearchFilters from "@/components/SearchFilters";
import { Prisma } from "@prisma/client";

export const metadata: Metadata = {
  title: "Business Directory — African Businesses in Berlin",
  description:
    "Browse all African-owned businesses in Berlin by category, location, and more.",
};

interface PageProps {
  searchParams: {
    q?: string;
    category?: string;
    featured?: string;
    page?: string;
  };
}

const PER_PAGE = 12;

export default async function DirectoryPage({ searchParams }: PageProps) {
  const page = Number(searchParams.page ?? 1);
  const skip = (page - 1) * PER_PAGE;

  const categories = await prisma.category.findMany({ orderBy: { name: "asc" } });

  const where: Prisma.BusinessWhereInput = {
    status: "APPROVED",
    ...(searchParams.q && {
      OR: [
        { name: { contains: searchParams.q, mode: "insensitive" } },
        { description: { contains: searchParams.q, mode: "insensitive" } },
        { address: { contains: searchParams.q, mode: "insensitive" } },
      ],
    }),
    ...(searchParams.category && {
      category: { slug: searchParams.category },
    }),
    ...(searchParams.featured === "true" && { featured: true }),
  };

  const [businesses, total] = await Promise.all([
    prisma.business.findMany({
      where,
      include: { category: true, owner: { select: { id: true, name: true, email: true } } },
      orderBy: [{ featured: "desc" }, { createdAt: "desc" }],
      take: PER_PAGE,
      skip,
    }),
    prisma.business.count({ where }),
  ]);

  const totalPages = Math.ceil(total / PER_PAGE);

  return (
    <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
      <div className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900">Business Directory</h1>
        <p className="text-gray-500 mt-1">{total} businesses found</p>
      </div>

      <SearchFilters categories={categories} />

      {businesses.length === 0 ? (
        <div className="text-center py-20">
          <p className="text-gray-400 text-xl">No businesses found.</p>
          <p className="text-gray-400 mt-2">Try adjusting your search or filters.</p>
        </div>
      ) : (
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6 mt-8">
          {businesses.map((biz) => (
            <BusinessCard key={biz.id} business={biz} />
          ))}
        </div>
      )}

      {/* Pagination */}
      {totalPages > 1 && (
        <div className="flex justify-center gap-2 mt-12">
          {Array.from({ length: totalPages }, (_, i) => i + 1).map((p) => (
            <a
              key={p}
              href={`/directory?page=${p}${searchParams.q ? `&q=${searchParams.q}` : ""}${searchParams.category ? `&category=${searchParams.category}` : ""}`}
              className={`px-4 py-2 rounded-lg border font-medium ${
                p === page
                  ? "bg-brand-500 text-white border-brand-500"
                  : "border-gray-300 text-gray-700 hover:bg-gray-50"
              }`}
            >
              {p}
            </a>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 17. app/business/[id]/page.tsx

```typescript
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import Image from "next/image";
import Link from "next/link";
import { prisma } from "@/lib/prisma";

interface PageProps {
  params: { id: string };
}

async function getBusiness(id: string) {
  return prisma.business.findFirst({
    where: { OR: [{ id }, { slug: id }], status: "APPROVED" },
    include: { category: true, owner: { select: { id: true, name: true, email: true } } },
  });
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const biz = await getBusiness(params.id);
  if (!biz) return { title: "Not Found" };
  return {
    title: `${biz.name} — ${biz.category.name} in Berlin`,
    description: biz.description.slice(0, 160),
  };
}

export default async function BusinessPage({ params }: PageProps) {
  const business = await getBusiness(params.id);
  if (!business) notFound();

  // Increment view count (fire-and-forget)
  prisma.business.update({
    where: { id: business.id },
    data: { viewCount: { increment: 1 } },
  }).catch(() => {});

  return (
    <div className="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
      {/* Breadcrumb */}
      <nav className="text-sm text-gray-500 mb-6">
        <Link href="/" className="hover:text-brand-500">Home</Link>
        {" / "}
        <Link href="/directory" className="hover:text-brand-500">Directory</Link>
        {" / "}
        <span className="text-gray-900">{business.name}</span>
      </nav>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-10">
        {/* Main Content */}
        <div className="lg:col-span-2">
          {/* Images */}
          {business.images.length > 0 ? (
            <div className="aspect-video relative rounded-2xl overflow-hidden mb-8">
              <Image
                src={business.images[0]}
                alt={business.name}
                fill
                className="object-cover"
              />
            </div>
          ) : (
            <div className="aspect-video bg-gradient-to-br from-brand-100 to-afro-gold/20 rounded-2xl flex items-center justify-center mb-8">
              <span className="text-6xl">🏢</span>
            </div>
          )}

          {business.images.length > 1 && (
            <div className="grid grid-cols-3 gap-2 mb-8">
              {business.images.slice(1, 4).map((img, i) => (
                <div key={i} className="aspect-square relative rounded-xl overflow-hidden">
                  <Image src={img} alt={`${business.name} photo ${i + 2}`} fill className="object-cover" />
                </div>
              ))}
            </div>
          )}

          {/* Details */}
          <div className="flex items-center gap-3 mb-4">
            {business.featured && (
              <span className="badge bg-afro-gold/10 text-afro-gold">⭐ Featured</span>
            )}
            <span className="badge bg-brand-50 text-brand-700">{business.category.name}</span>
          </div>

          <h1 className="text-3xl font-bold text-gray-900 mb-4">{business.name}</h1>

          <p className="text-gray-400 text-sm mb-6 flex items-center gap-1">
            📍 {business.address}, {business.city}
          </p>

          <div className="prose max-w-none text-gray-700">
            <p>{business.description}</p>
          </div>
        </div>

        {/* Sidebar */}
        <div className="space-y-4">
          <div className="bg-gray-50 rounded-2xl p-6 border border-gray-100">
            <h3 className="font-semibold text-gray-900 mb-4">Contact Business</h3>
            <div className="space-y-3">
              {business.phone && (
                <a
                  href={`tel:${business.phone}`}
                  className="flex items-center gap-3 text-gray-700 hover:text-brand-500"
                >
                  <span>📞</span>
                  <span>{business.phone}</span>
                </a>
              )}

              {business.whatsapp && (
                <a
                  href={`https://wa.me/${business.whatsapp.replace(/\D/g, "")}`}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="flex items-center justify-center gap-2 w-full bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl transition-colors"
                >
                  <span>💬</span> Chat on WhatsApp
                </a>
              )}

              {business.website && (
                <a
                  href={business.website}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="flex items-center gap-3 text-brand-500 hover:text-brand-600"
                >
                  <span>🌐</span>
                  <span className="truncate">{business.website}</span>
                </a>
              )}
            </div>
          </div>

          <div className="bg-gray-50 rounded-2xl p-6 border border-gray-100">
            <h3 className="font-semibold text-gray-900 mb-3">Location</h3>
            <p className="text-gray-600">📍 {business.address}</p>
            <p className="text-gray-500 text-sm">{business.city}, Germany</p>
          </div>

          <div className="text-center text-xs text-gray-400">
            {business.viewCount} views
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## 18. app/blog/page.tsx

```typescript
import type { Metadata } from "next";
import Link from "next/link";
import Image from "next/image";
import { prisma } from "@/lib/prisma";

export const metadata: Metadata = {
  title: "Blog — African Diaspora Stories, Tips & Business Insights",
  description:
    "Read articles, guides, and stories for African entrepreneurs and the diaspora community in Germany.",
};

export default async function BlogPage() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    orderBy: { createdAt: "desc" },
  });

  return (
    <div className="max-w-5xl mx-auto px-4 py-12">
      <div className="mb-10">
        <h1 className="text-3xl font-bold text-gray-900">AfroHub Blog</h1>
        <p className="text-gray-500 mt-2">
          Stories, guides, and insights for the African diaspora in Germany.
        </p>
      </div>

      {posts.length === 0 ? (
        <p className="text-gray-400 text-center py-20">No posts published yet.</p>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
          {posts.map((post) => (
            <Link
              key={post.id}
              href={`/blog/${post.slug}`}
              className="group card"
            >
              {post.coverImage && (
                <div className="aspect-video relative overflow-hidden">
                  <Image
                    src={post.coverImage}
                    alt={post.title}
                    fill
                    className="object-cover group-hover:scale-105 transition-transform duration-300"
                  />
                </div>
              )}
              <div className="p-6">
                <div className="flex flex-wrap gap-2 mb-3">
                  {post.tags.slice(0, 3).map((tag) => (
                    <span key={tag} className="badge bg-brand-50 text-brand-700">
                      {tag}
                    </span>
                  ))}
                </div>
                <h2 className="text-xl font-semibold text-gray-900 group-hover:text-brand-600 mb-2">
                  {post.title}
                </h2>
                <p className="text-gray-500 text-sm line-clamp-3">{post.excerpt}</p>
                <p className="text-xs text-gray-400 mt-4">
                  {new Date(post.createdAt).toLocaleDateString("en-DE", {
                    year: "numeric",
                    month: "long",
                    day: "numeric",
                  })}
                </p>
              </div>
            </Link>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 19. app/blog/[slug]/page.tsx

```typescript
import type { Metadata } from "next";
import { notFound } from "next/navigation";
import Image from "next/image";
import { prisma } from "@/lib/prisma";

interface PageProps {
  params: { slug: string };
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const post = await prisma.post.findUnique({ where: { slug: params.slug } });
  if (!post) return { title: "Not Found" };
  return {
    title: post.title,
    description: post.excerpt,
    keywords: post.tags,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: post.coverImage ? [post.coverImage] : [],
    },
  };
}

export default async function BlogPostPage({ params }: PageProps) {
  const post = await prisma.post.findUnique({
    where: { slug: params.slug, published: true },
  });
  if (!post) notFound();

  return (
    <article className="max-w-3xl mx-auto px-4 py-12">
      {post.coverImage && (
        <div className="aspect-video relative rounded-2xl overflow-hidden mb-8">
          <Image src={post.coverImage} alt={post.title} fill className="object-cover" />
        </div>
      )}
      <div className="flex flex-wrap gap-2 mb-4">
        {post.tags.map((tag) => (
          <span key={tag} className="badge bg-brand-50 text-brand-700">{tag}</span>
        ))}
      </div>
      <h1 className="text-4xl font-bold text-gray-900 mb-4 leading-tight">
        {post.title}
      </h1>
      <p className="text-gray-400 text-sm mb-10">
        {new Date(post.createdAt).toLocaleDateString("en-DE", {
          year: "numeric",
          month: "long",
          day: "numeric",
        })}
      </p>
      <div
        className="prose prose-lg max-w-none text-gray-700 leading-relaxed"
        dangerouslySetInnerHTML={{ __html: post.content }}
      />
    </article>
  );
}
```

---

## 20. app/dashboard/page.tsx

```typescript
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";
import Link from "next/link";
import { authOptions } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import DashboardLayout from "@/components/DashboardLayout";

export default async function DashboardPage() {
  const session = await getServerSession(authOptions);
  if (!session) redirect("/login");

  const businesses = await prisma.business.findMany({
    where: { ownerId: session.user.id },
    include: { category: true },
    orderBy: { createdAt: "desc" },
  });

  const subscription = await prisma.subscription.findUnique({
    where: { userId: session.user.id },
  });

  const isFeaturedEligible =
    subscription?.status === "ACTIVE";

  return (
    <DashboardLayout>
      <div className="space-y-8">
        {/* Header */}
        <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-4">
          <div>
            <h1 className="text-2xl font-bold text-gray-900">My Listings</h1>
            <p className="text-gray-500 mt-1">
              Manage your business listings on AfroHubMarket
            </p>
          </div>
          <Link href="/dashboard/create" className="btn-primary">
            + Add New Listing
          </Link>
        </div>

        {/* Subscription Banner */}
        {!isFeaturedEligible && (
          <div className="bg-gradient-to-r from-brand-50 to-afro-gold/10 border border-brand-200 rounded-xl p-5 flex flex-col sm:flex-row items-start sm:items-center justify-between gap-4">
            <div>
              <p className="font-semibold text-brand-800">⭐ Upgrade to Featured</p>
              <p className="text-brand-600 text-sm mt-0.5">
                Get your business shown at the top of all searches for €20/month.
              </p>
            </div>
            <a href="/api/stripe" className="btn-primary whitespace-nowrap">
              Upgrade — €20/mo
            </a>
          </div>
        )}

        {/* Stats */}
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          {[
            { label: "Total Listings", value: businesses.length },
            { label: "Total Views", value: businesses.reduce((s, b) => s + b.viewCount, 0) },
            { label: "WhatsApp Clicks", value: businesses.reduce((s, b) => s + b.whatsappClicks, 0) },
            { label: "Featured", value: businesses.filter(b => b.featured).length },
          ].map((stat) => (
            <div key={stat.label} className="bg-white rounded-xl border border-gray-100 p-5 text-center shadow-sm">
              <p className="text-3xl font-bold text-brand-500">{stat.value}</p>
              <p className="text-gray-500 text-sm mt-1">{stat.label}</p>
            </div>
          ))}
        </div>

        {/* Listings Table */}
        {businesses.length === 0 ? (
          <div className="card text-center py-16">
            <p className="text-gray-400 text-lg mb-4">No listings yet</p>
            <Link href="/dashboard/create" className="btn-primary">
              Create your first listing
            </Link>
          </div>
        ) : (
          <div className="card overflow-x-auto">
            <table className="w-full text-sm">
              <thead className="bg-gray-50 border-b border-gray-100">
                <tr>
                  {["Business", "Category", "Status", "Views", "Featured", "Actions"].map(
                    (h) => (
                      <th key={h} className="text-left px-6 py-4 text-gray-500 font-medium">
                        {h}
                      </th>
                    )
                  )}
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-50">
                {businesses.map((biz) => (
                  <tr key={biz.id} className="hover:bg-gray-50">
                    <td className="px-6 py-4 font-medium text-gray-900">{biz.name}</td>
                    <td className="px-6 py-4 text-gray-500">{biz.category.name}</td>
                    <td className="px-6 py-4">
                      <span
                        className={`badge ${
                          biz.status === "APPROVED"
                            ? "bg-green-50 text-green-700"
                            : biz.status === "PENDING"
                            ? "bg-yellow-50 text-yellow-700"
                            : "bg-red-50 text-red-700"
                        }`}
                      >
                        {biz.status}
                      </span>
                    </td>
                    <td className="px-6 py-4 text-gray-500">{biz.viewCount}</td>
                    <td className="px-6 py-4">
                      {biz.featured ? (
                        <span className="badge bg-afro-gold/10 text-afro-gold">⭐ Yes</span>
                      ) : (
                        <span className="text-gray-400">—</span>
                      )}
                    </td>
                    <td className="px-6 py-4 flex gap-3">
                      <Link
                        href={`/dashboard/edit/${biz.id}`}
                        className="text-brand-500 hover:text-brand-700 font-medium"
                      >
                        Edit
                      </Link>
                      <Link
                        href={`/business/${biz.slug}`}
                        className="text-gray-400 hover:text-gray-600 font-medium"
                      >
                        View
                      </Link>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </div>
    </DashboardLayout>
  );
}
```

---

## 21. app/dashboard/create/page.tsx

```typescript
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";
import { authOptions } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import DashboardLayout from "@/components/DashboardLayout";
import BusinessForm from "@/components/BusinessForm";

export default async function CreateListingPage() {
  const session = await getServerSession(authOptions);
  if (!session) redirect("/login");

  const categories = await prisma.category.findMany({
    orderBy: { name: "asc" },
  });

  return (
    <DashboardLayout>
      <div className="max-w-2xl">
        <h1 className="text-2xl font-bold text-gray-900 mb-2">Add New Listing</h1>
        <p className="text-gray-500 mb-8">
          Fill in the details below. Your listing will be reviewed within 24 hours.
        </p>
        <BusinessForm categories={categories} />
      </div>
    </DashboardLayout>
  );
}
```

---

## 22. app/admin/page.tsx

```typescript
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";
import { authOptions } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import DashboardLayout from "@/components/DashboardLayout";

export default async function AdminPage() {
  const session = await getServerSession(authOptions);
  if (!session || session.user.role !== "ADMIN") redirect("/");

  const [pendingBusinesses, allUsers, allBusinesses] = await Promise.all([
    prisma.business.findMany({
      where: { status: "PENDING" },
      include: { category: true, owner: { select: { id: true, name: true, email: true } } },
      orderBy: { createdAt: "asc" },
    }),
    prisma.user.findMany({ orderBy: { createdAt: "desc" } }),
    prisma.business.count(),
  ]);

  return (
    <DashboardLayout admin>
      <div className="space-y-10">
        <div>
          <h1 className="text-2xl font-bold text-gray-900">Admin Panel</h1>
          <p className="text-gray-500 mt-1">Manage listings, users, and platform settings</p>
        </div>

        {/* Stats */}
        <div className="grid grid-cols-2 md:grid-cols-3 gap-4">
          {[
            { label: "Total Users", value: allUsers.length },
            { label: "Total Listings", value: allBusinesses },
            { label: "Pending Review", value: pendingBusinesses.length },
          ].map((s) => (
            <div key={s.label} className="bg-white rounded-xl border p-5 text-center shadow-sm">
              <p className="text-3xl font-bold text-brand-500">{s.value}</p>
              <p className="text-gray-500 text-sm mt-1">{s.label}</p>
            </div>
          ))}
        </div>

        {/* Pending Listings */}
        <div>
          <h2 className="text-xl font-semibold text-gray-900 mb-4">
            Pending Approvals ({pendingBusinesses.length})
          </h2>
          {pendingBusinesses.length === 0 ? (
            <p className="text-gray-400">All caught up! No pending listings.</p>
          ) : (
            <div className="space-y-4">
              {pendingBusinesses.map((biz) => (
                <div key={biz.id} className="card p-5 flex flex-col sm:flex-row sm:items-center justify-between gap-4">
                  <div>
                    <p className="font-semibold text-gray-900">{biz.name}</p>
                    <p className="text-gray-500 text-sm">{biz.category.name} • {biz.address}</p>
                    <p className="text-gray-400 text-xs mt-1">
                      By {biz.owner.name ?? biz.owner.email} •{" "}
                      {new Date(biz.createdAt).toLocaleDateString()}
                    </p>
                  </div>
                  <div className="flex gap-3">
                    <form action={`/api/businesses/${biz.id}/approve`} method="POST">
                      <button type="submit" className="btn-primary bg-green-500 hover:bg-green-600">
                        ✓ Approve
                      </button>
                    </form>
                    <form action={`/api/businesses/${biz.id}/reject`} method="POST">
                      <button type="submit" className="btn-secondary text-red-600 border-red-200 hover:bg-red-50">
                        ✗ Reject
                      </button>
                    </form>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>

        {/* Users */}
        <div>
          <h2 className="text-xl font-semibold text-gray-900 mb-4">
            Users ({allUsers.length})
          </h2>
          <div className="card overflow-x-auto">
            <table className="w-full text-sm">
              <thead className="bg-gray-50 border-b">
                <tr>
                  {["Name", "Email", "Role", "Joined", "Actions"].map((h) => (
                    <th key={h} className="text-left px-6 py-4 text-gray-500 font-medium">{h}</th>
                  ))}
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-50">
                {allUsers.map((user) => (
                  <tr key={user.id} className="hover:bg-gray-50">
                    <td className="px-6 py-4 font-medium">{user.name ?? "—"}</td>
                    <td className="px-6 py-4 text-gray-500">{user.email}</td>
                    <td className="px-6 py-4">
                      <span className={`badge ${
                        user.role === "ADMIN" ? "bg-red-50 text-red-700" :
                        user.role === "BUSINESS_OWNER" ? "bg-brand-50 text-brand-700" :
                        "bg-gray-100 text-gray-600"
                      }`}>{user.role}</span>
                    </td>
                    <td className="px-6 py-4 text-gray-400 text-xs">
                      {new Date(user.createdAt).toLocaleDateString()}
                    </td>
                    <td className="px-6 py-4">
                      <form action={`/api/admin/delete-user/${user.id}`} method="POST">
                        <button type="submit" className="text-red-400 hover:text-red-600 text-xs">
                          Delete
                        </button>
                      </form>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </DashboardLayout>
  );
}
```

---

## 23. app/(auth)/login/page.tsx

```typescript
"use client";

import { signIn } from "next-auth/react";
import { useRouter } from "next/navigation";
import { useState } from "react";
import Link from "next/link";

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError("");
    const res = await signIn("credentials", {
      email,
      password,
      redirect: false,
    });
    setLoading(false);
    if (res?.error) {
      setError("Invalid email or password");
    } else {
      router.push("/dashboard");
    }
  }

  return (
    <div className="min-h-[80vh] flex items-center justify-center px-4">
      <div className="w-full max-w-md">
        <div className="text-center mb-8">
          <h1 className="text-3xl font-bold text-gray-900">Welcome back</h1>
          <p className="text-gray-500 mt-2">Sign in to your AfroHubMarket account</p>
        </div>

        <div className="card p-8">
          <button
            onClick={() => signIn("google", { callbackUrl: "/dashboard" })}
            className="w-full flex items-center justify-center gap-3 border border-gray-300 rounded-lg py-3 font-medium hover:bg-gray-50 transition-colors mb-6"
          >
            <svg className="w-5 h-5" viewBox="0 0 24 24">
              <path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
              <path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
              <path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
              <path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
            </svg>
            Continue with Google
          </button>

          <div className="flex items-center gap-4 mb-6">
            <div className="flex-1 h-px bg-gray-200" />
            <span className="text-gray-400 text-sm">or</span>
            <div className="flex-1 h-px bg-gray-200" />
          </div>

          {error && (
            <div className="bg-red-50 border border-red-200 text-red-700 text-sm px-4 py-3 rounded-lg mb-4">
              {error}
            </div>
          )}

          <form onSubmit={handleSubmit} className="space-y-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Email</label>
              <input
                type="email"
                className="input"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                required
                placeholder="you@example.com"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">Password</label>
              <input
                type="password"
                className="input"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
                placeholder="••••••••"
              />
            </div>
            <button type="submit" disabled={loading} className="btn-primary w-full">
              {loading ? "Signing in..." : "Sign In"}
            </button>
          </form>

          <p className="text-center text-gray-500 text-sm mt-6">
            Don't have an account?{" "}
            <Link href="/register" className="text-brand-500 hover:text-brand-600 font-medium">
              Create one free
            </Link>
          </p>
        </div>
      </div>
    </div>
  );
}
```

---

## 24. app/(auth)/register/page.tsx

```typescript
"use client";

import { useState } from "react";
import { signIn } from "next-auth/react";
import { useRouter } from "next/navigation";
import Link from "next/link";

export default function RegisterPage() {
  const router = useRouter();
  const [form, setForm] = useState({ name: "", email: "", password: "" });
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError("");

    const res = await fetch("/api/auth/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    });

    const data = await res.json();

    if (!res.ok) {
      setError(data.error ?? "Registration failed");
      setLoading(false);
      return;
    }

    await signIn("credentials", {
      email: form.email,
      password: form.password,
      callbackUrl: "/dashboard",
    });
  }

  return (
    <div className="min-h-[80vh] flex items-center justify-center px-4">
      <div className="w-full max-w-md">
        <div className="text-center mb-8">
          <h1 className="text-3xl font-bold text-gray-900">Create account</h1>
          <p className="text-gray-500 mt-2">Join the AfroHubMarket community</p>
        </div>

        <div className="card p-8">
          <button
            onClick={() => signIn("google", { callbackUrl: "/dashboard" })}
            className="w-full flex items-center justify-center gap-3 border border-gray-300 rounded-lg py-3 font-medium hover:bg-gray-50 mb-6"
          >
            <span>Continue with Google</span>
          </button>

          <div className="flex items-center gap-4 mb-6">
            <div className="flex-1 h-px bg-gray-200" />
            <span className="text-gray-400 text-sm">or</span>
            <div className="flex-1 h-px bg-gray-200" />
          </div>

          {error && (
            <div className="bg-red-50 border border-red-200 text-red-700 text-sm px-4 py-3 rounded-lg mb-4">
              {error}
            </div>
          )}

          <form onSubmit={handleSubmit} className="space-y-4">
            {(["name", "email", "password"] as const).map((f) => (
              <div key={f}>
                <label className="block text-sm font-medium text-gray-700 mb-1 capitalize">
                  {f}
                </label>
                <input
                  type={f === "password" ? "password" : f === "email" ? "email" : "text"}
                  className="input"
                  value={form[f]}
                  onChange={(e) => setForm((p) => ({ ...p, [f]: e.target.value }))}
                  required
                />
              </div>
            ))}
            <button type="submit" disabled={loading} className="btn-primary w-full">
              {loading ? "Creating account..." : "Create Free Account"}
            </button>
          </form>

          <p className="text-center text-gray-500 text-sm mt-6">
            Already have an account?{" "}
            <Link href="/login" className="text-brand-500 hover:text-brand-600 font-medium">
              Sign in
            </Link>
          </p>
        </div>
      </div>
    </div>
  );
}
```

---

## 25. app/api/auth/[...nextauth]/route.ts

```typescript
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

---

## 26. app/api/auth/register/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import bcrypt from "bcryptjs";
import { prisma } from "@/lib/prisma";
import { registerSchema } from "@/lib/validations";

export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const parsed = registerSchema.safeParse(body);

    if (!parsed.success) {
      return NextResponse.json(
        { error: parsed.error.errors[0].message },
        { status: 400 }
      );
    }

    const { name, email, password } = parsed.data;

    const existing = await prisma.user.findUnique({ where: { email } });
    if (existing) {
      return NextResponse.json(
        { error: "An account with this email already exists" },
        { status: 409 }
      );
    }

    const hashed = await bcrypt.hash(password, 12);

    await prisma.user.create({
      data: { name, email, password: hashed },
    });

    return NextResponse.json({ success: true }, { status: 201 });
  } catch {
    return NextResponse.json({ error: "Server error" }, { status: 500 });
  }
}
```

---

## 27. app/api/businesses/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import { businessSchema } from "@/lib/validations";

function toSlug(name: string): string {
  return name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/(^-|-$)/g, "");
}

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const q = searchParams.get("q");
  const category = searchParams.get("category");
  const featured = searchParams.get("featured");

  const businesses = await prisma.business.findMany({
    where: {
      status: "APPROVED",
      ...(q && {
        OR: [
          { name: { contains: q, mode: "insensitive" } },
          { description: { contains: q, mode: "insensitive" } },
        ],
      }),
      ...(category && { category: { slug: category } }),
      ...(featured === "true" && { featured: true }),
    },
    include: { category: true, owner: { select: { id: true, name: true, email: true } } },
    orderBy: [{ featured: "desc" }, { createdAt: "desc" }],
  });

  return NextResponse.json(businesses);
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  try {
    const body = await req.json();
    const parsed = businessSchema.safeParse(body);

    if (!parsed.success) {
      return NextResponse.json(
        { error: parsed.error.errors[0].message },
        { status: 400 }
      );
    }

    const { name, ...rest } = parsed.data;
    let slug = toSlug(name);

    // Ensure unique slug
    const exists = await prisma.business.findUnique({ where: { slug } });
    if (exists) slug = `${slug}-${Date.now()}`;

    // Update user role to BUSINESS_OWNER if not already
    if (session.user.role === "USER") {
      await prisma.user.update({
        where: { id: session.user.id },
        data: { role: "BUSINESS_OWNER" },
      });
    }

    const business = await prisma.business.create({
      data: {
        name,
        slug,
        ...rest,
        ownerId: session.user.id,
        status: "PENDING",
      },
      include: { category: true },
    });

    return NextResponse.json(business, { status: 201 });
  } catch (err) {
    console.error(err);
    return NextResponse.json({ error: "Server error" }, { status: 500 });
  }
}
```

---

## 28. app/api/businesses/[id]/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { prisma } from "@/lib/prisma";
import { businessSchema } from "@/lib/validations";

interface Params { params: { id: string } }

export async function PUT(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const biz = await prisma.business.findUnique({ where: { id: params.id } });
  if (!biz) return NextResponse.json({ error: "Not found" }, { status: 404 });

  if (biz.ownerId !== session.user.id && session.user.role !== "ADMIN") {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  const body = await req.json();
  const parsed = businessSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.errors[0].message }, { status: 400 });
  }

  const updated = await prisma.business.update({
    where: { id: params.id },
    data: { ...parsed.data, status: "PENDING" }, // re-review on edit
    include: { category: true },
  });

  return NextResponse.json(updated);
}

export async function DELETE(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const biz = await prisma.business.findUnique({ where: { id: params.id } });
  if (!biz) return NextResponse.json({ error: "Not found" }, { status: 404 });

  if (biz.ownerId !== session.user.id && session.user.role !== "ADMIN") {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  await prisma.business.delete({ where: { id: params.id } });
  return NextResponse.json({ success: true });
}

// Admin approve
export async function PATCH(req: NextRequest, { params }: Params) {
  const session = await getServerSession(authOptions);
  if (!session || session.user.role !== "ADMIN") {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  const { action } = await req.json();

  const updated = await prisma.business.update({
    where: { id: params.id },
    data: {
      status: action === "approve" ? "APPROVED" : "REJECTED",
    },
  });

  return NextResponse.json(updated);
}
```

---

## 29. app/api/stripe/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { stripe, FEATURED_PRICE_ID, getOrCreateStripeCustomer } from "@/lib/stripe";

export async function GET(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session) return NextResponse.redirect(new URL("/login", req.url));

  const customerId = await getOrCreateStripeCustomer(
    session.user.id,
    session.user.email!,
    session.user.name
  );

  const checkoutSession = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: "subscription",
    payment_method_types: ["card"],
    line_items: [{ price: FEATURED_PRICE_ID, quantity: 1 }],
    success_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?upgraded=true`,
    cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard`,
    metadata: { userId: session.user.id },
  });

  return NextResponse.redirect(checkoutSession.url!);
}
```

---

## 30. app/api/stripe/webhook/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import { stripe } from "@/lib/stripe";
import { prisma } from "@/lib/prisma";
import Stripe from "stripe";

export const config = { api: { bodyParser: false } };

export async function POST(req: NextRequest) {
  const body = await req.text();
  const sig = req.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  const getCustomerUserId = async (customerId: string) => {
    const sub = await prisma.subscription.findUnique({
      where: { stripeCustomerId: customerId },
    });
    return sub?.userId;
  };

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.Checkout.Session;
      const userId = session.metadata?.userId;
      if (!userId) break;

      const subscription = await stripe.subscriptions.retrieve(
        session.subscription as string
      );

      await prisma.subscription.upsert({
        where: { userId },
        update: {
          stripeSubscriptionId: subscription.id,
          stripePriceId: subscription.items.data[0].price.id,
          status: "ACTIVE",
          currentPeriodEnd: new Date(subscription.current_period_end * 1000),
        },
        create: {
          userId,
          stripeCustomerId: session.customer as string,
          stripeSubscriptionId: subscription.id,
          stripePriceId: subscription.items.data[0].price.id,
          status: "ACTIVE",
          currentPeriodEnd: new Date(subscription.current_period_end * 1000),
        },
      });
      break;
    }

    case "invoice.payment_succeeded": {
      const invoice = event.data.object as Stripe.Invoice;
      const userId = await getCustomerUserId(invoice.customer as string);
      if (!userId) break;
      const sub = await stripe.subscriptions.retrieve(invoice.subscription as string);
      await prisma.subscription.update({
        where: { userId },
        data: {
          status: "ACTIVE",
          currentPeriodEnd: new Date(sub.current_period_end * 1000),
        },
      });
      break;
    }

    case "customer.subscription.deleted": {
      const sub = event.data.object as Stripe.Subscription;
      const userId = await getCustomerUserId(sub.customer as string);
      if (!userId) break;

      await prisma.subscription.update({
        where: { userId },
        data: { status: "CANCELED" },
      });

      // Unfeature all their listings
      await prisma.business.updateMany({
        where: { ownerId: userId },
        data: { featured: false },
      });
      break;
    }

    case "invoice.payment_failed": {
      const invoice = event.data.object as Stripe.Invoice;
      const userId = await getCustomerUserId(invoice.customer as string);
      if (userId) {
        await prisma.subscription.update({
          where: { userId },
          data: { status: "PAST_DUE" },
        });
      }
      break;
    }
  }

  return NextResponse.json({ received: true });
}
```

---

## 31. app/api/upload/route.ts

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";
import { v2 as cloudinary } from "cloudinary";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const formData = await req.formData();
  const file = formData.get("file") as File;

  if (!file) return NextResponse.json({ error: "No file provided" }, { status: 400 });

  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  const result = await new Promise<{ secure_url: string }>((resolve, reject) => {
    cloudinary.uploader
      .upload_stream(
        {
          folder: "afrohubmarket",
          transformation: [{ width: 1200, height: 800, crop: "fill", quality: "auto" }],
        },
        (err, result) => {
          if (err || !result) reject(err);
          else resolve(result as { secure_url: string });
        }
      )
      .end(buffer);
  });

  return NextResponse.json({ url: result.secure_url });
}
```

---

## 32. components/Navbar.tsx

```typescript
"use client";

import { useSession, signOut } from "next-auth/react";
import Link from "next/link";
import { useState } from "react";

export default function Navbar() {
  const { data: session } = useSession();
  const [mobileOpen, setMobileOpen] = useState(false);

  return (
    <nav className="bg-white border-b border-gray-100 sticky top-0 z-50">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div className="flex items-center justify-between h-16">
          {/* Logo */}
          <Link href="/" className="flex items-center gap-2">
            <span className="text-2xl">🌍</span>
            <span className="font-bold text-xl text-gray-900">
              AfroHub<span className="text-brand-500">Market</span>
            </span>
          </Link>

          {/* Desktop Nav */}
          <div className="hidden md:flex items-center gap-6">
            <Link href="/directory" className="text-gray-600 hover:text-brand-500 font-medium">
              Directory
            </Link>
            <Link href="/blog" className="text-gray-600 hover:text-brand-500 font-medium">
              Blog
            </Link>
            {session ? (
              <>
                <Link href="/dashboard" className="text-gray-600 hover:text-brand-500 font-medium">
                  Dashboard
                </Link>
                {session.user.role === "ADMIN" && (
                  <Link href="/admin" className="text-red-500 hover:text-red-600 font-medium">
                    Admin
                  </Link>
                )}
                <button
                  onClick={() => signOut({ callbackUrl: "/" })}
                  className="btn-secondary"
                >
                  Sign Out
                </button>
              </>
            ) : (
              <>
                <Link href="/login" className="btn-secondary">Sign In</Link>
                <Link href="/register" className="btn-primary">Get Listed Free</Link>
              </>
            )}
          </div>

          {/* Mobile toggle */}
          <button
            className="md:hidden p-2 rounded-lg hover:bg-gray-100"
            onClick={() => setMobileOpen(!mobileOpen)}
          >
            <span className="sr-only">Menu</span>
            <div className="space-y-1.5">
              <span className="block w-6 h-0.5 bg-gray-600" />
              <span className="block w-6 h-0.5 bg-gray-600" />
              <span className="block w-6 h-0.5 bg-gray-600" />
            </div>
          </button>
        </div>
      </div>

      {/* Mobile Menu */}
      {mobileOpen && (
        <div className="md:hidden bg-white border-t border-gray-100 px-4 py-4 space-y-3">
          <Link href="/directory" className="block text-gray-700 font-medium py-2">Directory</Link>
          <Link href="/blog" className="block text-gray-700 font-medium py-2">Blog</Link>
          {session ? (
            <>
              <Link href="/dashboard" className="block text-gray-700 font-medium py-2">Dashboard</Link>
              <button
                onClick={() => signOut()}
                className="block text-gray-700 font-medium py-2 w-full text-left"
              >
                Sign Out
              </button>
            </>
          ) : (
            <>
              <Link href="/login" className="block text-gray-700 font-medium py-2">Sign In</Link>
              <Link href="/register" className="btn-primary block text-center">Get Listed Free</Link>
            </>
          )}
        </div>
      )}
    </nav>
  );
}
```

---

## 33. components/Footer.tsx

```typescript
import Link from "next/link";

export default function Footer() {
  return (
    <footer className="bg-gray-900 text-gray-300 mt-auto">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
        <div className="grid grid-cols-1 md:grid-cols-4 gap-8 mb-8">
          <div>
            <div className="flex items-center gap-2 mb-3">
              <span className="text-2xl">🌍</span>
              <span className="font-bold text-xl text-white">
                AfroHub<span className="text-brand-500">Market</span>
              </span>
            </div>
            <p className="text-gray-400 text-sm leading-relaxed">
              The African diaspora marketplace in Berlin and Germany.
            </p>
          </div>

          {[
            {
              title: "Discover",
              links: [
                { href: "/directory", label: "Business Directory" },
                { href: "/directory?category=restaurant", label: "Restaurants" },
                { href: "/directory?category=hair-beauty", label: "Hair & Beauty" },
                { href: "/blog", label: "Blog" },
              ],
            },
            {
              title: "For Business",
              links: [
                { href: "/dashboard/create", label: "List Your Business" },
                { href: "/api/stripe", label: "Get Featured — €20/mo" },
                { href: "/dashboard", label: "Business Dashboard" },
              ],
            },
            {
              title: "Company",
              links: [
                { href: "/about", label: "About AfroHub" },
                { href: "/contact", label: "Contact" },
                { href: "/privacy", label: "Privacy Policy" },
                { href: "/terms", label: "Terms of Service" },
              ],
            },
          ].map((section) => (
            <div key={section.title}>
              <h4 className="font-semibold text-white mb-3">{section.title}</h4>
              <ul className="space-y-2">
                {section.links.map((l) => (
                  <li key={l.href}>
                    <Link href={l.href} className="text-gray-400 hover:text-white text-sm transition-colors">
                      {l.label}
                    </Link>
                  </li>
                ))}
              </ul>
            </div>
          ))}
        </div>

        <div className="border-t border-gray-800 pt-6 flex flex-col sm:flex-row justify-between items-center gap-4">
          <p className="text-gray-500 text-sm">
            © {new Date().getFullYear()} AfroHubMarket. Built for the African diaspora 🌍
          </p>
          <div className="flex gap-4">
            {["Berlin", "Hamburg", "Frankfurt", "Munich"].map((city) => (
              <Link
                key={city}
                href={`/directory?city=${city}`}
                className="text-gray-500 hover:text-gray-300 text-xs"
              >
                {city}
              </Link>
            ))}
          </div>
        </div>
      </div>
    </footer>
  );
}
```

---

## 34. components/BusinessCard.tsx

```typescript
import Link from "next/link";
import Image from "next/image";
import { BusinessWithCategory } from "@/types";

interface Props {
  business: BusinessWithCategory;
  compact?: boolean;
}

export default function BusinessCard({ business, compact = false }: Props) {
  return (
    <Link href={`/business/${business.slug}`} className="card group block">
      {/* Image */}
      <div className={`relative overflow-hidden bg-gray-100 ${compact ? "aspect-video" : "aspect-[4/3]"}`}>
        {business.images[0] ? (
          <Image
            src={business.images[0]}
            alt={business.name}
            fill
            className="object-cover group-hover:scale-105 transition-transform duration-300"
          />
        ) : (
          <div className="w-full h-full flex items-center justify-center bg-gradient-to-br from-brand-50 to-brand-100">
            <span className="text-4xl">🏢</span>
          </div>
        )}
        {business.featured && (
          <span className="absolute top-3 left-3 badge bg-afro-gold text-white">
            ⭐ Featured
          </span>
        )}
      </div>

      {/* Content */}
      <div className={`p-${compact ? "4" : "5"}`}>
        <div className="flex items-center justify-between mb-1">
          <span className="badge bg-brand-50 text-brand-700 text-xs">
            {business.category.name}
          </span>
          {business.whatsapp && (
            <span className="text-green-500 text-xs font-medium">WhatsApp ✓</span>
          )}
        </div>
        <h3 className="font-semibold text-gray-900 group-hover:text-brand-600 mt-2 line-clamp-1">
          {business.name}
        </h3>
        <p className="text-gray-500 text-sm mt-1 line-clamp-2">{business.description}</p>
        <p className="text-gray-400 text-xs mt-3 flex items-center gap-1">
          📍 {business.address}
        </p>
      </div>
    </Link>
  );
}
```

---

## 35. components/SearchFilters.tsx

```typescript
"use client";

import { useRouter, useSearchParams } from "next/navigation";
import { useState, useTransition } from "react";
import { Category } from "@prisma/client";

interface Props {
  categories?: Category[];
  compact?: boolean;
}

export default function SearchFilters({ categories = [], compact = false }: Props) {
  const router = useRouter();
  const params = useSearchParams();
  const [, startTransition] = useTransition();
  const [q, setQ] = useState(params.get("q") ?? "");

  function handleSearch(e: React.FormEvent) {
    e.preventDefault();
    const p = new URLSearchParams(params.toString());
    if (q) p.set("q", q); else p.delete("q");
    p.delete("page");
    startTransition(() => router.push(`/directory?${p.toString()}`));
  }

  function handleCategory(slug: string) {
    const p = new URLSearchParams(params.toString());
    if (slug) p.set("category", slug); else p.delete("category");
    p.delete("page");
    startTransition(() => router.push(`/directory?${p.toString()}`));
  }

  return (
    <div className={compact ? "" : "mb-8"}>
      <form onSubmit={handleSearch} className="flex gap-3">
        <input
          type="text"
          className="input flex-1"
          placeholder="Search businesses, services..."
          value={q}
          onChange={(e) => setQ(e.target.value)}
        />
        <button type="submit" className="btn-primary px-8">
          Search
        </button>
      </form>

      {!compact && categories.length > 0 && (
        <div className="flex flex-wrap gap-2 mt-4">
          <button
            onClick={() => handleCategory("")}
            className={`px-4 py-1.5 rounded-full border text-sm font-medium transition-colors ${
              !params.get("category")
                ? "bg-brand-500 text-white border-brand-500"
                : "border-gray-300 text-gray-600 hover:border-brand-500"
            }`}
          >
            All
          </button>
          {categories.map((cat) => (
            <button
              key={cat.id}
              onClick={() => handleCategory(cat.slug)}
              className={`px-4 py-1.5 rounded-full border text-sm font-medium transition-colors ${
                params.get("category") === cat.slug
                  ? "bg-brand-500 text-white border-brand-500"
                  : "border-gray-300 text-gray-600 hover:border-brand-500"
              }`}
            >
              {cat.name}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 36. components/BusinessForm.tsx

```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Category } from "@prisma/client";

interface Props {
  categories: Category[];
  initialData?: Record<string, unknown>;
  businessId?: string;
}

export default function BusinessForm({ categories, initialData, businessId }: Props) {
  const router = useRouter();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const [uploadingImages, setUploadingImages] = useState(false);

  const [form, setForm] = useState({
    name: (initialData?.name as string) ?? "",
    description: (initialData?.description as string) ?? "",
    categoryId: (initialData?.categoryId as string) ?? "",
    address: (initialData?.address as string) ?? "",
    city: (initialData?.city as string) ?? "Berlin",
    phone: (initialData?.phone as string) ?? "",
    whatsapp: (initialData?.whatsapp as string) ?? "",
    website: (initialData?.website as string) ?? "",
    images: (initialData?.images as string[]) ?? [],
  });

  function set(key: string, value: string) {
    setForm((p) => ({ ...p, [key]: value }));
  }

  async function handleImageUpload(e: React.ChangeEvent<HTMLInputElement>) {
    const files = Array.from(e.target.files ?? []);
    if (!files.length) return;
    setUploadingImages(true);

    const uploaded: string[] = [];
    for (const file of files) {
      const fd = new FormData();
      fd.append("file", file);
      const res = await fetch("/api/upload", { method: "POST", body: fd });
      const data = await res.json();
      if (data.url) uploaded.push(data.url);
    }

    setForm((p) => ({ ...p, images: [...p.images, ...uploaded] }));
    setUploadingImages(false);
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError("");

    const res = await fetch(businessId ? `/api/businesses/${businessId}` : "/api/businesses", {
      method: businessId ? "PUT" : "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    });

    const data = await res.json();
    setLoading(false);

    if (!res.ok) {
      setError(data.error ?? "Failed to save listing");
      return;
    }

    router.push("/dashboard");
    router.refresh();
  }

  const FIELDS = [
    { key: "name", label: "Business Name", required: true, placeholder: "e.g. Lagos Kitchen Berlin" },
    { key: "address", label: "Address", required: true, placeholder: "e.g. Sonnenallee 45, Berlin" },
    { key: "city", label: "City", required: true, placeholder: "Berlin" },
    { key: "phone", label: "Phone Number", placeholder: "+49 30 1234567" },
    { key: "whatsapp", label: "WhatsApp Number", placeholder: "+4917612345678" },
    { key: "website", label: "Website", placeholder: "https://yourwebsite.com" },
  ];

  return (
    <form onSubmit={handleSubmit} className="space-y-6">
      {error && (
        <div className="bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded-lg text-sm">
          {error}
        </div>
      )}

      {/* Text Fields */}
      {FIELDS.map((f) => (
        <div key={f.key}>
          <label className="block text-sm font-medium text-gray-700 mb-1">
            {f.label} {f.required && <span className="text-red-500">*</span>}
          </label>
          <input
            type="text"
            className="input"
            value={(form as Record<string, string>)[f.key]}
            onChange={(e) => set(f.key, e.target.value)}
            required={f.required}
            placeholder={f.placeholder}
          />
        </div>
      ))}

      {/* Category */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Category <span className="text-red-500">*</span>
        </label>
        <select
          className="input"
          value={form.categoryId}
          onChange={(e) => set("categoryId", e.target.value)}
          required
        >
          <option value="">Select a category...</option>
          {categories.map((cat) => (
            <option key={cat.id} value={cat.id}>
              {cat.name}
            </option>
          ))}
        </select>
      </div>

      {/* Description */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Description <span className="text-red-500">*</span>
        </label>
        <textarea
          className="input min-h-[120px] resize-y"
          value={form.description}
          onChange={(e) => set("description", e.target.value)}
          required
          minLength={20}
          placeholder="Tell customers about your business, what you offer, opening hours..."
        />
      </div>

      {/* Image Upload */}
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-1">
          Photos
        </label>
        <input
          type="file"
          accept="image/*"
          multiple
          onChange={handleImageUpload}
          className="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-sm file:font-semibold file:bg-brand-50 file:text-brand-700 hover:file:bg-brand-100"
        />
        {uploadingImages && <p className="text-sm text-gray-400 mt-2">Uploading images...</p>}
        {form.images.length > 0 && (
          <div className="flex flex-wrap gap-2 mt-3">
            {form.images.map((img, i) => (
              <div key={i} className="relative group">
                {/* eslint-disable-next-line @next/next/no-img-element */}
                <img src={img} alt="" className="w-20 h-20 object-cover rounded-lg" />
                <button
                  type="button"
                  onClick={() => setForm((p) => ({ ...p, images: p.images.filter((_, j) => j !== i) }))}
                  className="absolute -top-2 -right-2 bg-red-500 text-white rounded-full w-5 h-5 text-xs flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity"
                >
                  ×
                </button>
              </div>
            ))}
          </div>
        )}
      </div>

      <button
        type="submit"
        disabled={loading || uploadingImages}
        className="btn-primary w-full text-lg py-3"
      >
        {loading ? "Saving..." : businessId ? "Update Listing" : "Submit for Review"}
      </button>

      <p className="text-gray-400 text-sm text-center">
        Listings are reviewed within 24 hours before going live.
      </p>
    </form>
  );
}
```

---

## 37. components/DashboardLayout.tsx

```typescript
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";
import Link from "next/link";
import { authOptions } from "@/lib/auth";

interface Props {
  children: React.ReactNode;
  admin?: boolean;
}

const NAV = [
  { href: "/dashboard", label: "My Listings", icon: "🏢" },
  { href: "/dashboard/create", label: "Add Listing", icon: "+" },
  { href: "/api/stripe", label: "Get Featured", icon: "⭐" },
];

const ADMIN_NAV = [
  { href: "/admin", label: "Admin Panel", icon: "⚙️" },
  ...NAV,
];

export default async function DashboardLayout({ children, admin }: Props) {
  const session = await getServerSession(authOptions);
  if (!session) redirect("/login");

  const links = session.user.role === "ADMIN" ? ADMIN_NAV : NAV;

  return (
    <div className="min-h-screen bg-gray-50">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
        <div className="flex flex-col lg:flex-row gap-8">
          {/* Sidebar */}
          <aside className="w-full lg:w-56 shrink-0">
            <div className="bg-white rounded-xl border border-gray-100 p-4">
              <div className="mb-4 pb-4 border-b border-gray-100">
                <p className="text-sm font-semibold text-gray-900 truncate">
                  {session.user.name ?? session.user.email}
                </p>
                <p className="text-xs text-gray-400 capitalize mt-0.5">
                  {session.user.role.toLowerCase().replace("_", " ")}
                </p>
              </div>
              <nav className="space-y-1">
                {links.map((item) => (
                  <Link
                    key={item.href}
                    href={item.href}
                    className="flex items-center gap-3 px-3 py-2.5 rounded-lg text-sm font-medium text-gray-600 hover:bg-brand-50 hover:text-brand-600 transition-colors"
                  >
                    <span>{item.icon}</span>
                    {item.label}
                  </Link>
                ))}
              </nav>
            </div>
          </aside>

          {/* Main */}
          <main className="flex-1 min-w-0">{children}</main>
        </div>
      </div>
    </div>
  );
}
```

---

## 38. prisma/seed.ts

```typescript
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
  // Seed categories
  const categories = [
    { name: "Restaurant", slug: "restaurant", icon: "🍽️" },
    { name: "Hair & Beauty", slug: "hair-beauty", icon: "💇" },
    { name: "Grocery & Shop", slug: "grocery", icon: "🛒" },
    { name: "Logistics", slug: "logistics", icon: "📦" },
    { name: "Finance", slug: "finance", icon: "💰" },
    { name: "Legal & Advice", slug: "legal", icon: "⚖️" },
    { name: "Events", slug: "events", icon: "🎉" },
    { name: "Health", slug: "health", icon: "🏥" },
    { name: "Fashion", slug: "fashion", icon: "👗" },
    { name: "Technology", slug: "technology", icon: "💻" },
  ];

  for (const cat of categories) {
    await prisma.category.upsert({
      where: { slug: cat.slug },
      update: {},
      create: cat,
    });
  }

  // Seed admin user
  const adminPassword = await bcrypt.hash("Admin@123", 12);
  await prisma.user.upsert({
    where: { email: "admin@afrohubmarket.com" },
    update: {},
    create: {
      name: "AfroHub Admin",
      email: "admin@afrohubmarket.com",
      password: adminPassword,
      role: "ADMIN",
    },
  });

  console.log("✅ Seed complete: categories and admin user created");
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## 39. SETUP INSTRUCTIONS

### Step 1 — Clone and Install

```bash
git clone <your-repo>
cd afrohubmarket
npm install
```

### Step 2 — Environment Variables

```bash
cp .env.example .env.local
# Fill in all values in .env.local
```

**Key values to fill:**

- `NEXTAUTH_SECRET` → run: `openssl rand -base64 32`
- `DATABASE_URL` → from Neon (neon.tech) or Supabase dashboard
- `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` → Google Cloud Console → OAuth 2.0
- `STRIPE_SECRET_KEY` → Stripe Dashboard → Developers → API keys
- `STRIPE_FEATURED_PRICE_ID` → Create a product in Stripe at €20/month, copy the Price ID
- `STRIPE_WEBHOOK_SECRET` → from `stripe listen` (see Step 6)
- `CLOUDINARY_*` → Cloudinary dashboard → Settings → Access Keys

### Step 3 — Database Setup

```bash
# Push schema to database
npx prisma db push

# Or run migrations (recommended for production)
npx prisma migrate dev --name init

# Seed categories and admin user
npm run db:seed
```

### Step 4 — Run Development Server

```bash
npm run dev
# App runs at http://localhost:3000
```

### Step 5 — Stripe Local Testing

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Forward webhooks to localhost
stripe listen --forward-to localhost:3000/api/stripe/webhook

# Copy the webhook secret printed and set STRIPE_WEBHOOK_SECRET in .env.local
```

### Step 6 — Google OAuth Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project → Enable Google+ API
3. OAuth 2.0 Credentials → Add authorized redirect URI:
   `http://localhost:3000/api/auth/callback/google`

### Step 7 — Cloudinary Setup

1. Create free account at cloudinary.com
2. Note your Cloud Name, API Key, API Secret
3. Add them to `.env.local`

### Step 8 — Deploy to Vercel

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Set environment variables in Vercel dashboard (same as .env.local)
# Add production Google OAuth redirect URI and Stripe webhook for production URL
```

---

## 40. ADMIN ACCESS

After seeding, log in with:
- **Email:** admin@afrohubmarket.com
- **Password:** Admin@123

Change the password immediately in the database or add a profile settings page.

---

## 41. KEY ARCHITECTURAL DECISIONS

| Decision | Choice | Why |
|---|---|---|
| Auth strategy | JWT session | Works seamlessly with App Router and edge |
| Image storage | Cloudinary | Free tier, auto-optimization, CDN |
| Payment | Stripe Checkout | Hosted, PCI compliant, no card data touches your server |
| DB adapter | Neon/Supabase | Serverless PostgreSQL, free tier, Vercel-native |
| Approval flow | Manual admin review | Prevents spam, builds quality directory |
| Featured listings | Stripe subscription | Recurring revenue, auto-disable on cancellation |
| Slug generation | name → lowercase-slug | SEO-friendly URLs like `/business/lagos-kitchen-berlin` |
```
