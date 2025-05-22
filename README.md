# Periskope Chat Application

This is a real-time chat application. It includes user authentication, real-time messaging, and a responsive, responsive UI.

## Features

* **User Authentication:**
  * Sign-up with email and password.
  * Sign-in for existing users.
  * Secure session management.
  * Automatic user profile creation in the database upon sign-up.
* **Real-time Chat:**
  * View a list of your chats.
  * Open a chat to see conversation history.
  * Send messages stored in the database.
  * Receive messages in real-time without refreshing.
* **Route Protection:**
  * Unauthenticated users are redirected to `/login`.
  * Authenticated users accessing `/login` are redirected to the chat interface (`/`).
* **Pixel-Perfect UI:** Designed to match provided design specifications.

## Tech Stack

* **Frontend:**
  * [Next.js](https://nextjs.org/) (v15+ with App Router)
  * [React](https://reactjs.org/)
  * [TypeScript](https://www.typescriptlang.org/)
  * [Tailwind CSS](https://tailwindcss.com/)
  * [shadcn/ui](https://ui.shadcn.com/)
* **Backend & Database:**
  * [Supabase](https://supabase.io/)
    * Authentication
    * PostgreSQL Database
    * Realtime Subscriptions
* **Package Manager:**
  * [npm](https://www.npmjs.com/)

## Prerequisites

* Node.js (v18 or later)
* npm Package Manager
* Supabase account and project

## Getting Started

Follow these instructions to set up and run the project locally.

### 1. Clone the Repository

```bash
git clone git@github.com:your-username/projectname.git
cd projectname
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Environment Variables

Create a `.env` file in the root of the `periskope` directory and add your Supabase project URL and Anon Key:

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_ANON_KEY=your_supabase_anon_key
```

Replace `your_supabase_project_url` and `your_supabase_anon_key` with your actual Supabase credentials. You can find these in your Supabase project settings under "API".

### 4. Supabase Setup

You need to run SQL scripts in your Supabase SQL Editor to set up the necessary tables, functions, triggers, and Row Level Security (RLS) policies.

**a. Database Schema & RLS:**

Execute the following SQL in your Supabase project's SQL Editor (under "Database"):

```sql
-- Create profiles table to store public user data
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE,
  full_name TEXT,
  avatar_url TEXT,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL
);

-- Function to automatically update 'updated_at' timestamp
CREATE OR REPLACE FUNCTION public.handle_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = timezone('utc'::text, now());
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger for profiles table
CREATE TRIGGER on_profiles_updated
  BEFORE UPDATE ON public.profiles
  FOR EACH ROW
  EXECUTE PROCEDURE public.handle_updated_at();

-- Create chats table
CREATE TABLE public.chats (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  last_message_id UUID, -- Nullable, to be updated by a trigger or function later
  created_by UUID REFERENCES public.profiles(id) ON DELETE SET NULL -- user who initiated the chat
);

-- Trigger for chats table
CREATE TRIGGER on_chats_updated
  BEFORE UPDATE ON public.chats
  FOR EACH ROW
  EXECUTE PROCEDURE public.handle_updated_at();

-- Create chat_participants table (junction table)
CREATE TABLE public.chat_participants (
  chat_id UUID REFERENCES public.chats(id) ON DELETE CASCADE,
  user_id UUID REFERENCES public.profiles(id) ON DELETE CASCADE,
  joined_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  PRIMARY KEY (chat_id, user_id)
);

-- Create messages table
CREATE TABLE public.messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id UUID REFERENCES public.chats(id) ON DELETE CASCADE NOT NULL,
  user_id UUID REFERENCES public.profiles(id) ON DELETE SET NULL, -- Set to NULL if user profile is deleted
  content TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT timezone('utc'::text, now()) NOT NULL,
  is_read BOOLEAN DEFAULT FALSE
);

-- Add foreign key constraint for last_message_id in chats table
-- This must be done after messages table is created
ALTER TABLE public.chats
ADD CONSTRAINT fk_last_message
FOREIGN KEY (last_message_id)
REFERENCES public.messages(id) ON DELETE SET NULL;

-- Enable Row Level Security for all tables
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chats ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chat_participants ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;

-- RLS Policies for profiles
CREATE POLICY "Public profiles are viewable by everyone."
  ON public.profiles FOR SELECT
  USING (true);

CREATE POLICY "Users can insert their own profile."
  ON public.profiles FOR INSERT
  WITH CHECK (auth.uid() = id);

CREATE POLICY "Users can update their own profile."
  ON public.profiles FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- RLS Policies for chats
CREATE POLICY "Users can view chats they are a participant in."
  ON public.chats FOR SELECT
  USING (EXISTS (
    SELECT 1
    FROM public.chat_participants cp
    WHERE cp.chat_id = chats.id AND cp.user_id = auth.uid()
  ));

CREATE POLICY "Authenticated users can create chats."
  ON public.chats FOR INSERT
  WITH CHECK (auth.role() = 'authenticated');
  -- Additional check: created_by should be the logged-in user
  -- WITH CHECK (auth.role() = 'authenticated' AND created_by = auth.uid()); -- Consider adding this if created_by is always auth.uid()

CREATE POLICY "Users can update chats they are a participant in (e.g., last_message_id)."
  ON public.chats FOR UPDATE
  USING (EXISTS (
    SELECT 1
    FROM public.chat_participants cp
    WHERE cp.chat_id = chats.id AND cp.user_id = auth.uid()
  ));

-- RLS Policies for chat_participants
CREATE POLICY "Users can view participants of chats they are in."
  ON public.chat_participants FOR SELECT
  USING (EXISTS (
    SELECT 1
    FROM public.chat_participants cp_self
    WHERE cp_self.chat_id = chat_participants.chat_id AND cp_self.user_id = auth.uid()
  ));

CREATE POLICY "Users can insert themselves or be added to chats by participants (or a chat creator)."
  ON public.chat_participants FOR INSERT
  WITH CHECK (auth.role() = 'authenticated');
  
-- RLS Policies for messages
CREATE POLICY "Users can view messages in chats they are a participant in."
  ON public.messages FOR SELECT
  USING (EXISTS (
    SELECT 1
    FROM public.chat_participants cp
    WHERE cp.chat_id = messages.chat_id AND cp.user_id = auth.uid()
  ));

CREATE POLICY "Users can insert messages into chats they are a participant in."
  ON public.messages FOR INSERT
  WITH CHECK (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1
      FROM public.chat_participants cp
      WHERE cp.chat_id = messages.chat_id AND cp.user_id = auth.uid()
    )
  );

-- Function to handle new user sign-ups and create a profile
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email) -- Assuming you want to store email, adjust if not
  VALUES (NEW.id, NEW.email);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger to call handle_new_user on new auth.users entry
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();
```

**b. Enable Realtime:**

Execute the following SQL to enable realtime updates for `messages`, `chats`, and `chat_participants`:

```sql
-- Add tables to the supabase_realtime publication
ALTER PUBLICATION supabase_realtime ADD TABLE public.messages;
ALTER PUBLICATION supabase_realtime ADD TABLE public.chats;
ALTER PUBLICATION supabase_realtime ADD TABLE public.chat_participants;

-- Ensure RLS is enforced for realtime. This is usually default but good to be explicit.
-- Check your Supabase project settings under API > Realtime to ensure "Enable Realtime" is on
-- and "Enable Row Level Security" for Realtime is also checked.
```

### 5. Run the Application

```bash
npm dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser to see the application.

## Key Components Explained

* **`lib/supabaseClient.ts`**:
  Initializes and exports the Supabase client using `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` environment variables. This client enables interactions with Supabase services (Authentication, Database, Realtime).

* **`components/auth-provider.tsx`**:
  Central to user authentication management, this component:
  * Provides user session data (`user`, `session`, `isLoading`) via React Context to child components.
  * Subscribes to Supabase's `onAuthStateChange` for real-time session updates.
  * Enforces route protection:
    * Redirects unauthenticated users from protected routes (e.g., `/`) to `/login`.
    * Redirects authenticated users from `/login` to `/`.
  * Exposes `signIn`, `signUp`, and `signOut` functions for authentication.

* **`app/login/page.tsx`**:
  Manages the UI and logic for user sign-up and sign-in, leveraging `AuthProvider` functions that invoke Supabase authentication methods.

* **`components/chat-interface.tsx`**:
  Core chat functionality component that:
  * Retrieves the logged-in user's chat list.
  * Renders chats in a sidebar (using `components/sidebar.tsx`).
  * Enables chat selection to view message history.
  * Fetches messages for the selected chat.
  * Includes an input field for sending new messages.
  * Uses Supabase Realtime subscriptions to update messages and chat list in real-time.

* **Database Schema & RLS**:
  * **`profiles`**: Stores public user data, tied to `auth.users`, auto-populated on sign-up via a trigger.
  * **`chats`**: Holds chat conversation metadata.
  * **`chat_participants`**: Junction table linking users to chats.
  * **`messages`**: Stores messages, linked to chats and users.
  * Row Level Security (RLS) policies ensure users only access/modify authorized data (e.g., view only their chats, send messages in participated chats).