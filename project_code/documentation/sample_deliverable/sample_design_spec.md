# Design Specification

> **Created by:** Agent_03_Designer  
> **Phase:** 2  
> **Input:** docs/user_stories.md, project_setup/step_3_design/

---

## Design System

### Colors
- **Primary:** #3B82F6 (Blue)
- **Secondary:** #10B981 (Green)
- **Error:** #EF4444 (Red)
- **Background:** #FFFFFF (White)
- **Text Primary:** #1F2937 (Dark Gray)
- **Text Secondary:** #6B7280 (Medium Gray)

### Typography
- **Heading 1:** 32px, Bold, Inter
- **Heading 2:** 24px, Semibold, Inter
- **Body:** 16px, Regular, Inter
- **Caption:** 14px, Regular, Inter

### Spacing
- **xs:** 4px
- **sm:** 8px
- **md:** 16px
- **lg:** 24px
- **xl:** 32px

---

## Screen: Registration

### Layout
- Centered card (max-width: 400px)
- White background with shadow
- Padding: 32px

### Components
1. **Logo** - Top center, 48px height
2. **Heading** - "Create Account", H1
3. **Email Input** - Full width, label "Email", placeholder "you@example.com"
4. **Password Input** - Full width, label "Password", type password, show/hide toggle
5. **Confirm Password Input** - Full width, label "Confirm Password"
6. **Submit Button** - Full width, primary color, "Sign Up"
7. **Link** - "Already have an account? Log in"

### Validation States
- **Error:** Red border, error message below input
- **Success:** Green checkmark icon
- **Loading:** Disabled button with spinner

---

## Screen: Login

### Layout
- Same card layout as Registration

### Components
1. **Logo** - Top center
2. **Heading** - "Welcome Back", H1
3. **Email Input**
4. **Password Input** with "Forgot password?" link
5. **Submit Button** - "Log In"
6. **Link** - "Don't have an account? Sign up"

---

## Screen: Dashboard

### Layout
- Full-width layout with sidebar

### Components
1. **Header** - Logo left, user menu right
2. **Sidebar** - Navigation menu (200px width)
3. **Main Content Area**
   - Profile card (top)
   - Recent activity list (below)

### Profile Card
- Avatar (64px circle)
- User email
- Member since date

### Activity List
- Each item: icon, description, timestamp
- Max 10 items
- "View all" link at bottom
