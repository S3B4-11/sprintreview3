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

