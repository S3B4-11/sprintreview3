# sprintreview3

## Código publicado por Diego / Necesita revisión.

## LoginActivity, archivo encargado del inicio de sesión

package com.api.practicasubo.login

import android.content.Intent
import android.os.Bundle
import android.view.ViewGroup
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import com.api.practicasubo.R
import com.api.practicasubo.core.Api
import com.api.practicasubo.core.Session
import com.api.practicasubo.main.MainActivity
import kotlinx.coroutines.runBlocking

class LoginActivity : AppCompatActivity() {

    var etUser: EditText? = null
    var etPass: EditText? = null
    var tvMsg: TextView? = null
    var btn: Button? = null

    private val auth: AuthService by lazy { Api.retrofit.create(AuthService::class.java) }

    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        setContentView(R.layout.activity_login)

        val root = window.decorView as ViewGroup
        val overlay = FrameLayout(this)
        overlay.setBackgroundColor(resources.getColor(R.color.brandBlue))
        val logo = ImageView(this)
        logo.setImageResource(R.drawable.ic_logo_flat)
        logo.scaleType = ImageView.ScaleType.CENTER_INSIDE
        overlay.addView(logo)
        root.addView(overlay)
        overlay.postDelayed({ root.removeView(overlay) }, 500)

        Session.load(this)
        if (!Session.token.isNullOrBlank()) {
            startActivity(Intent(this, MainActivity::class.java))
            finish()
            return
        }

        etUser = findViewById(R.id.etUser)
        etPass = findViewById(R.id.etPass)
        tvMsg = findViewById(R.id.tvMsg)
        btn = findViewById(R.id.btnLogin)

        findViewById<TextView>(R.id.tvForgot).setOnClickListener {
            startActivity(Intent(this, ForgotPasswordActivity::class.java))
        }

        btn!!.setOnClickListener {
            tvMsg!!.text = ""
            btn!!.isEnabled = false
            btn!!.text = "Ingresando..."

            val u = etUser!!.text.toString()
            val p = etPass!!.text.toString()

            runBlocking {
                try {
                    val res = auth.login(mapOf("username" to u, "password" to p))
                    Session.save(this@LoginActivity, res.token)
                    val r = res.user.role.lowercase()
                    Session.saveRole(this@LoginActivity, r)
                    Session.saveIdentity(this@LoginActivity, res.user.email ?: u, res.user.full_name ?: u)
                    startActivity(Intent(this@LoginActivity, MainActivity::class.java))
                    finish()
                } catch (e: Exception) {
                    tvMsg!!.text = "Error de login"
                    btn!!.isEnabled = true
                    btn!!.text = "Iniciar sesión"
                }
            }
        }
    }
}


Revision de sebastian 

package com.api.practicasubo.login

import android.content.Intent
import android.os.Bundle
import android.view.ViewGroup
import android.widget.*
import android.view.inputmethod.EditorInfo
import androidx.appcompat.app.AppCompatActivity
import androidx.core.widget.doOnTextChanged
import androidx.lifecycle.lifecycleScope
import com.api.practicasubo.R
import com.api.practicasubo.core.Api
import com.api.practicasubo.core.Session
import com.api.practicasubo.main.MainActivity
import com.google.android.material.button.MaterialButton
import kotlinx.coroutines.launch

class LoginActivity : AppCompatActivity() {

    private val auth: AuthService by lazy { Api.retrofit.create(AuthService::class.java) }

    private lateinit var etUser: EditText
    private lateinit var etPass: EditText
    private lateinit var tvMsg: TextView
    private lateinit var btnLogin: MaterialButton

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        addSimpleOverlay()

        Session.load(this)
        if (!Session.token.isNullOrBlank()) return goToMain()

        etUser = findViewById(R.id.etUser)
        etPass = findViewById(R.id.etPass)
        tvMsg  = findViewById(R.id.tvMsg)
        btnLogin = findViewById(R.id.btnLogin)

        findViewById<TextView>(R.id.tvForgot).setOnClickListener {
            startActivity(Intent(this, ForgotPasswordActivity::class.java))
        }

        val updateEnabled = {
            btnLogin.isEnabled = etUser.text?.isNotEmpty() == true && etPass.text?.isNotEmpty() == true
        }
        etUser.doOnTextChanged { _, _, _, _ -> updateEnabled() }
        etPass.doOnTextChanged { _, _, _, _ -> updateEnabled() }
        updateEnabled()

        etPass.setOnEditorActionListener { _, id, _ ->
            if (id == EditorInfo.IME_ACTION_DONE && btnLogin.isEnabled) { btnLogin.performClick(); true } else false
        }

        btnLogin.setOnClickListener { doLogin() }
    }

    private fun doLogin() {
        val username = etUser.text.toString().trim()
        val password = etPass.text.toString().trim()
        if (username.isEmpty() || password.isEmpty()) {
            toast("Completa usuario y contraseña"); return
        }

        setLoading(true)
        lifecycleScope.launch {
            val body = mapOf("username" to username, "password" to password)
            try {
                val res = auth.login(body)
                Session.save(this@LoginActivity, res.token)
                Session.saveRole(this@LoginActivity, normalizeRole(res.user.role))
                val display = res.user.full_name?.takeIf { it.isNotBlank() } ?: res.user.username
                val email = res.user.email ?: res.user.username
                Session.saveIdentity(this@LoginActivity, email, display)
                goToMain()
            } catch (e: Exception) {
                tvMsg.text = "No pudimos iniciar sesión. Revisa las credenciales e inténtalo de nuevo."
                setLoading(false)
            }
        }
    }

    private fun normalizeRole(raw: String?): String = when (raw?.lowercase()) {
        "estudiante" -> "alumno"
        "empresa"    -> "empresa"
        "admin"      -> "admin"
        else         -> raw?.lowercase().orEmpty()
    }

    private fun setLoading(loading: Boolean) {
        btnLogin.isEnabled = !loading
        btnLogin.text = if (loading) getString(R.string.logging_in) else getString(R.string.action_login)
        tvMsg.text = ""
    }

    private fun goToMain() {
        startActivity(Intent(this, MainActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK)
        })
        finish()
    }

    private fun addSimpleOverlay() {
        val root = window.decorView as ViewGroup
        val overlay = FrameLayout(this).apply { setBackgroundColor(getColor(R.color.brandBlue)) }
        val logo = ImageView(this).apply {
            setImageResource(R.drawable.ic_logo_flat)
            scaleType = ImageView.ScaleType.CENTER_INSIDE
            val pad = (48 * resources.displayMetrics.density).toInt()
            setPadding(pad, pad, pad, pad)
        }
        overlay.addView(logo)
        root.addView(overlay)
        overlay.postDelayed({ root.removeView(overlay) }, 600)
    }

    private fun toast(msg: String) = Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()
}
