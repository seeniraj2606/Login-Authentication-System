# Login-Authentication-System
import React, { createContext, useContext, useState, useEffect } from 'react';
// Assuming a service for API calls is defined
import { loginUser, logoutUser } from './authService'; 

// 1. Create the Context
const AuthContext = createContext(null);

// 2. Custom Hook to use the context
export const useAuth = () => useContext(AuthContext);

// 3. Context Provider Component
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  // Check for a token/user on app load
  useEffect(() => {
    const storedUser = localStorage.getItem('fe_user');
    if (storedUser) {
      setUser(JSON.parse(storedUser));
    }
    setLoading(false);
  }, []);

  // Login Function
  const login = async (email, password) => {
    try {
      const { data } = await loginUser(email, password); // API call
      
      // Store user info and token (for this FE example, we'll use local storage)
      localStorage.setItem('fe_user', JSON.stringify(data.user));
      localStorage.setItem('fe_token', data.token); 
      
      setUser(data.user);
      return { success: true };
    } catch (error) {
      console.error("Login failed:", error);
      return { success: false, message: error.response?.data?.message || "Login failed. Check credentials." };
    }
  };

  // Logout Function
  const logout = () => {
    logoutUser(); // Optional: call a backend endpoint to clear server-side session
    localStorage.removeItem('fe_user');
    localStorage.removeItem('fe_token');
    setUser(null);
  };

  const value = {
    user,
    isAuthenticated: !!user,
    loading,
    login,
    logout,
  };

  // Render children only after checking stored state
  if (loading) return <div>Loading Application...</div>; 

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Example usage in App.js:
/*
<AuthProvider>
  <AppRoutes />
</AuthProvider>
*/
import React, { useState } from 'react';
import { useNavigate, Navigate } from 'react-router-dom';
import { useAuth } from './AuthContext';

// Simple style/utility functions (in a real IBM FE app, you'd use Carbon Design Components)
const LoginContainer = ({ children }) => <div style={{ padding: '40px', maxWidth: '400px', margin: '50px auto', border: '1px solid #ccc' }}>{children}</div>;
const Input = (props) => <input style={{ width: '100%', padding: '10px', margin: '8px 0', boxSizing: 'border-box' }} {...props} />;
const Button = (props) => <button style={{ width: '100%', padding: '10px', background: '#0f62fe', color: 'white', border: 'none', cursor: 'pointer' }} {...props} />;
const ErrorMsg = ({ children }) => <p style={{ color: 'red', fontSize: '0.9em' }}>{children}</p>;

export default function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const { login, isAuthenticated } = useAuth();
  const navigate = useNavigate();

  // If already authenticated, redirect to the dashboard
  if (isAuthenticated) {
    return <Navigate to="/dashboard" replace />;
  }

  const handleSubmit = async (e) => {
    e.preventDefault();
    setError(''); // Clear previous errors
    
    // Simple client-side validation
    if (!email || !password) {
      setError('Please enter both email and password.');
      return;
    }
    
    setIsSubmitting(true);

    const result = await login(email, password);
    
    if (result.success) {
      // Navigate to the protected route on success
      navigate('/dashboard'); 
    } else {
      // Show error message returned from the auth logic
      setError(result.message);
    }

    setIsSubmitting(false);
  };

  return (
    <LoginContainer>
      <h2>IBM FE Login System</h2>
      <form onSubmit={handleSubmit}>
        <div>
          <label htmlFor="email">Email</label>
          <Input 
            type="email" 
            id="email"
            value={email} 
            onChange={(e) => setEmail(e.target.value)} 
            placeholder="user@ibm.com"
            required
          />
        </div>
        <div>
          <label htmlFor="password">Password</label>
          <Input 
            type="password" 
            id="password"
            value={password} 
            onChange={(e) => setPassword(e.target.value)} 
            placeholder="********"
            required
          />
        </div>
        {error && <ErrorMsg>{error}</ErrorMsg>}
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Logging In...' : 'Log In'}
        </Button>
      </form>
    </LoginContainer>
  );
}
import React from 'react';
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from './AuthContext';

// Use this component to wrap routes that require a logged-in user
export default function PrivateRoute() {
  const { isAuthenticated, loading } = useAuth();

  if (loading) {
    return <div>Loading user session...</div>;
  }

  // If the user is authenticated, render the child route content
  // Otherwise, redirect them to the login page
  return isAuthenticated ? <Outlet /> : <Navigate to="/" replace />;
}
